# PAX 包管理器手册

PAX 是 XCX 的官方包管理器，直接集成在 `xcx` 二进制中。它管理依赖、项目脚手架与构建自动化。

## 项目配置：`project.pax`

每个 PAX 项目必须在其根目录拥有 `project.pax` 文件。它使用自定义声明式格式。

```pax
---
PAX Project Configuration
*---
/
    name :: "my_project",
    deps :: [
        "user/repo",
        "https://example.com/lib.xcx"
    ]
/
```

- **name**：项目的逻辑名称。
- **deps**：依赖列表。支持 GitHub 快捷方式（`user/repo`）与直接 URL。

## 命令参考

PAX 通过 `xcx pax <command>` 调用。

| 命令 | 说明 |
|--------------------------|------------------------------------------------------|
| `xcx pax new <name>` | 生成新项目结构。 |
| `xcx pax clone <package>` | 将已发布包下载为本地项目。 |
| `xcx pax install` | 将依赖获取到 `lib/` 目录。 |
| `xcx pax add <dep>` | 添加依赖并立即安装。 |
| `xcx pax remove <name>` | 从 `project.pax` 移除依赖。 |
| `xcx pax search <query>` | 在注册表中搜索可用包。 |
| `xcx pax run [path]` | 执行项目（入口：`src/main.xcx`）。 |

### `xcx pax clone <package>`

从 PAX 注册表下载已发布包到新本地目录，可直接运行或修改。

```sh
xcx pax clone snake_game
xcx pax clone beta/snake_game
```

- 参数为注册表中列出的包名（可选 `author/` 前缀）。
- 在当前工作目录创建以包名命名的目录。
- 复制完整项目结构：`project.pax`、`src/` 及 `files` 中声明的其他文件。
- **不会**自动安装依赖——克隆后需在目录内运行 `xcx pax install`。

```sh
xcx pax clone snake_game
cd snake_game
xcx pax install
xcx pax run
```

## 目录结构

标准 PAX 项目遵循以下布局：
- `project.pax`：配置。
- `src/`：源代码（主入口：`main.xcx`）。
- `lib/`：已下载依赖（由 PAX 管理）。
- `tests/`：项目特定测试。
