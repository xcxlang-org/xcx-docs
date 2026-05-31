# XCX 3.1 标准库与模块

## 内置模块

### crypto

加密工具。

| 方法                                | 返回值 | 说明                                    |
|-------------------------------------|--------|-----------------------------------------|
| `crypto.hash(data, "bcrypt")`       | `s`    | 使用 bcrypt 哈希密码                    |
| `crypto.hash(data, "argon2")`       | `s`    | 使用 argon2 哈希密码（推荐）            |
| `crypto.hash(data, "base64_encode")`| `s`    | 将二进制数据/字符串编码为 Base64 字符串 |
| `crypto.hash(data, "base64_decode")`| `s`    | 将 Base64 字符串解码为二进制/字符串     |
| `crypto.verify(password, hash, algo)` | `b`  | 密码与哈希匹配时返回 `true`             |
| `crypto.token(length)`              | `s`    | 生成指定长度的随机十六进制令牌          |

```xcx
s: hash  = crypto.hash(password, "bcrypt");
s: hash2 = crypto.hash(password, "argon2");
b: valid = crypto.verify(password, hash2, "argon2");
s: token = crypto.token(32);
```

### store（文件 I/O）

所有路径必须相对于项目根目录。**绝对路径**或路径遍历（`..`）将触发 `halt.fatal`。

| 方法                | 签名                      | 返回值    | 说明                                      |
|---------------------|---------------------------|-----------|-------------------------------------------|
| `store.write(p, c)` | `(s, s) → b`              | `b`       | 覆盖写入文件。必要时创建目录。            |
| `store.read(p)`     | `(s) → s`                 | `s`       | 返回文件内容；文件不存在时 `halt.fatal`。 |
| `store.append(p, c)`| `(s, s) → b`              | `b`       | 追加到文件。不存在则创建。                |
| `store.exists(p)`   | `(s) → b`                 | `b`       | 检查是否存在。无副作用。                  |
| `store.delete(p)`   | `(s) → b`                 | `b`       | 删除文件或目录（递归）。                  |
| `store.list(p)`     | `(s) → array:s`           | `array:s` | 返回文件和文件夹列表。                    |
| `store.isDir(p)`    | `(s) → b`                 | `b`       | 路径为目录时返回 `true`。                 |
| `store.size(p)`     | `(s) → i`                 | `i`       | 文件大小（字节）。                        |
| `store.mkdir(p)`    | `(s) → b`                 | `b`       | 创建目录（递归）。                        |
| `store.glob(pat)`   | `(s) → array:s`           | `array:s` | 返回匹配 glob 模式的文件。                |
| `store.zip(s, t)`   | `(s, s) → b`              | `b`       | 将源路径归档为目标 zip。                  |
| `store.unzip(z, d)` | `(s, s) → b`              | `b`       | 将 zip 解压到目标路径。                   |

```xcx
store.write("log.txt", "First line");
store.append("log.txt", "\nSecond line");
s: content = store.read("log.txt");
if (store.exists("lock.pid")) then;
    >! "Already running";
end;
```

### env

| 方法            | 签名           | 返回值    | 说明                                           |
|-----------------|----------------|-----------|------------------------------------------------|
| `env.get(name)` | `(s) → s`      | `s`       | 返回环境变量值；未设置时 `halt.error`          |
| `env.args()`    | `() → array:s` | `array:s` | 返回作为数组传入程序的 CLI 参数                |

```xcx
s: db_url = env.get("DATABASE_URL");

array:s: args = env.args();
for arg in args do;
    >! arg;
end;
```

### random

| 方法                              | 签名                | 返回值 | 说明                                                      |
|-----------------------------------|---------------------|--------|-----------------------------------------------------------|
| `random.choice from col`          | `(set/array) → T`   | `T`    | 从给定集合或数组中随机选取一个元素。                      |
| `random.int(min, max @step num)`  | `(i, i, @i) → i`    | `i`    | 在范围 `[min, max]` 内随机选取整数。默认步长：`1`。       |
| `random.float(min, max @step num)`| `(f, f, @f) → f`    | `f`    | 在范围 `[min, max]` 内随机选取浮点数。默认步长：`0.5`。   |


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

### date（模块）

```xcx
date: now = date.now();
```

---

## 模块系统

### include

将另一文件的代码合并到当前命名空间。

```xcx
include "utils.xcx";
include "math.xcx" as m;

m.PI;
m.sqrt(16.0);
```

无别名时——所有符号直接在当前命名空间中可用。有别名时——所有顶层符号带前缀：`alias.symbol`。

循环依赖在编译时会被检测并拒绝。
