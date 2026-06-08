# PAX 包管理器手册

PAX 是 XCX 的官方包管理器，直接集成在 `xcx` 二进制文件中。它负责管理依赖项、项目脚手架、注册表发布和构建自动化。

---

## 项目配置：`project.pax`

每个 PAX 项目必须在根目录中包含一个 `project.pax` 文件，使用自定义声明格式。

```pax
---
PAX Project Configuration
*---
/
    name        :: "my_project",
    version     :: "1.0.0",
    author      :: "yourname",
    description :: "A short description of the project",
    main        :: "src/main.xcx",
    tags  :: ["tag1", "tag2"],
    files :: ["src/main.xcx", "src/lib.xcx"],
    deps  :: [
        "somelib@1.0.0",
        "author/repo@latest",
        "https://example.com/lib.xcx"
    ]
/
```

### 字段说明

| 字段          | 必填 | 说明                                                                         |
|---------------|------|------------------------------------------------------------------------------|
| `name`        | 是   | 项目/包的名称。                                                               |
| `version`     | 是   | 语义化版本字符串（例如 `"1.0.0"`）。                                          |
| `author`      | 否   | 作者名称，显示在注册表中。                                                    |
| `description` | 否   | 项目的简短描述。                                                              |
| `main`        | 否   | 自定义入口点。省略时默认为 `src/main.xcx`。                                   |
| `tags`        | 否   | 用于注册表可发现性的标签列表。                                                |
| `files`       | 否   | 发布时包含的文件列表。省略时 PAX 会自动排除已知的非代码文件。                  |
| `deps`        | 否   | 依赖项列表。支持版本化名称、`author/repo` 简写和直接 URL。                    |

### 依赖格式

```pax
deps :: [
    "mylib@1.0.0",           --- 指定版本的注册表包
    "mylib@latest",          --- 最新版本的注册表包
    "author/mylib@2.1.0",    --- 带作者前缀
    "https://example.com/lib.xcx"  --- 直接 URL
]
```

---

## 锁定文件：`pax.lock`

PAX 在每次安装后自动维护 `pax.lock` 文件，记录每个已安装依赖项的确切解析版本和本地路径。

```json
{
  "mylib": { "version": "mylib@1.0.0", "file": "lib/mylib/" },
  "otherlib": { "version": "otherlib@latest", "file": "lib/otherlib/" }
}
```

- **不要**手动编辑 `pax.lock`。
- 将其提交到版本控制以实现可重现的构建。

---

## 目录结构

标准 PAX 项目的目录布局如下：

```
my_project/
├── project.pax       # 项目配置
├── pax.lock          # 自动生成的锁定文件
├── src/
│   └── main.xcx      # 主入口点
├── lib/              # 已安装的依赖项（由 PAX 管理）
└── tests/            # 项目专属测试
```

---

## 命令参考

PAX 通过 `xcx pax <command>` 调用。

| 命令                         | 说明                                           |
|------------------------------|------------------------------------------------|
| `xcx pax new <name>`         | 创建新项目结构。                               |
| `xcx pax install`            | 将所有依赖项下载到 `lib/`。                    |
| `xcx pax add <dep>`          | 添加依赖项并立即安装。                         |
| `xcx pax remove <name>`      | 从 `project.pax` 中移除依赖项。                |
| `xcx pax clone <package>`    | 将已发布的包下载为本地项目。                   |
| `xcx pax run [path]`         | 执行项目入口点。                               |
| `xcx pax search <query>`     | 在注册表中搜索可用包。                         |
| `xcx pax login <token>`      | 存储注册表身份验证令牌。                       |
| `xcx pax logout`             | 移除已存储的身份验证令牌。                     |
| `xcx pax whoami`             | 验证当前注册表账户。                           |
| `xcx pax publish`            | 将项目发布到 PAX 注册表。                      |

---

## 命令详情

### `xcx pax new <name>`

使用标准目录布局创建新项目。

```sh
xcx pax new my_project
```

创建：
```
my_project/
├── project.pax
├── src/
│   └── main.xcx
└── README.md
```

---

### `xcx pax install`

读取 `project.pax`，解析所有 `deps` 并下载到 `lib/`。跳过已存在的包（按名称）。

```sh
xcx pax install
```

- 仅安装依赖项自身 `files` 字段中声明的文件。
- 如果依赖项没有 `files` 字段，PAX 自动排除 `README.md`、`LICENSE`、`tests/`、`docs/`、`.git*` 和 `project.pax`。
- 每次成功获取后更新 `pax.lock`。

---

### `xcx pax add <dep>`

向 `project.pax` 添加依赖项并立即运行安装。

```sh
xcx pax add mylib@1.0.0
xcx pax add author/mylib@latest
```

---

### `xcx pax remove <name>`

从 `project.pax` 中移除依赖项条目。

```sh
xcx pax remove mylib
```

> **注意：** `lib/` 文件夹和 `pax.lock` 条目不会自动清理。请重新运行 `xcx pax install` 或手动删除。

---

### `xcx pax clone <package>`

将已发布的包从 PAX 注册表下载为完整的本地项目，可直接运行或修改。

```sh
xcx pax clone snake_game
xcx pax clone author/snake_game
```

- 在当前工作目录中创建以包命名的目录。
- 复制完整项目结构，包括 `project.pax`、`src/` 和所有声明的文件。
- **不会**自动安装依赖项。

**典型工作流程：**
```sh
xcx pax clone snake_game
cd snake_game
xcx pax install
xcx pax run
```

---

### `xcx pax run [path]`

执行项目。入口点解析顺序：

1. 作为参数提供的路径（例如 `xcx pax run other/main.xcx`）
2. `project.pax` 中的 `main` 字段
3. 默认：`src/main.xcx`

```sh
xcx pax run
xcx pax run src/alt.xcx
```

---

### `xcx pax search <query>`

在 PAX 注册表中搜索匹配查询的包。

```sh
xcx pax search json
```

输出格式：
```
- json_utils (v1.2.0) by alice
- fast_json (v0.9.1) by bob
```

---

### `xcx pax login <token>`

将注册表身份验证令牌本地存储在 `.pax_token` 中。

```sh
xcx pax login your_token_here
```

---

### `xcx pax logout`

移除已存储的身份验证令牌。

```sh
xcx pax logout
```

---

### `xcx pax whoami`

验证当前注册表会话并打印账户信息。

```sh
xcx pax whoami
# Account: alice [developer]
```

---

### `xcx pax publish`

将当前项目发布到 PAX 注册表。需要登录。

```sh
xcx pax publish
```

- 从 `project.pax` 读取元数据（`name`、`version`、`description`）。
- 扫描并上传所有项目文件（排除 `.zip`、`.pax_token`、`.git`）。
- 向注册表发送压缩存档和文件清单。

> **注意：** 每次发布前请在 `project.pax` 中更新 `version`。注册表可能拒绝重复版本。

---

## 注册表配置

PAX 默认连接到 `pax.xcxlang.com`。可通过项目根目录中的 `.pax_config` 文件覆盖：

```json
{
  "registry": "https://my-custom-registry.example.com"
}
```

- `localhost` 和 `127.0.0.1` 自动使用 HTTP。
- 所有其他主机使用 HTTPS。
