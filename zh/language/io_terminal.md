# XCX 3.1 I/O 与系统命令

## 控制台输出（`>!`）

`>!` 运算符将值打印到 `stdout`，并在末尾追加换行。

```xcx
>! "Hello";
>! 42;
>! "Path: " + path;
```

支持**转义序列**，如 `\n`（换行）、`\t`（制表符）和 `\r`（回车）。

## 用户输入（`>?`）

`>?` 运算符从 `stdin` 读取一行，并尝试将其解析到目标变量。

```xcx
i: age;
>! "Enter age:";
>? age;
```

## 延迟（`@wait`）

暂停 VM 执行指定的毫秒数。

```xcx
@wait 1000; --- Waits for 1 second
```

> `@wait` 是**阻塞**操作，不会 yield 纤程。

## 原始输入（`input`）

`input` 模块允许在不等待 Enter 的情况下读取单个按键。

### 方法

| 调用                | 返回值 | 说明                               |
|---------------------|--------|------------------------------------|
| `input.key()`       | `s`    | 若有按键则返回，否则返回 `""`      |
| `input.key() @wait` | `s`    | 等待按键并返回                     |
| `input.ready()`     | `b`    | 缓冲区中有按键等待时为 `true`      |

### 按键常量

| 值              | 按键         |
|-----------------|--------------|
| `"UP"`          | 上箭头       |
| `"DOWN"`        | 下箭头       |
| `"LEFT"`        | 左箭头       |
| `"RIGHT"`       | 右箭头       |
| `"ENTER"`       | Enter        |
| `"ESC"`         | Escape       |
| `"BACKSPACE"`   | Backspace    |
| `"TAB"`         | Tab          |
| `"F1"` ... `"F12"` | F1–F12    |
| `"KEY_CTRL_C"`  | Ctrl+C       |
| `"KEY_CTRL_Z"`  | Ctrl+Z       |
| `"KEY_CTRL_S"`  | Ctrl+S       |

普通字符直接返回：`"a"`、`"Z"`、`"5"`、`" "`。

### 示例

```xcx
s: k = input.key();
if (k == "UP") then;
    y = y - 1;
end;

--- wait for specific key
s: confirm = input.key() @wait;
if (confirm == "q") then;
    return;
end;
```

## 终端命令（`.terminal`）

直接与系统环境或当前进程交互。

| 指令                   | 说明                                   |
|------------------------|----------------------------------------|
| `.terminal !clear`     | 清屏                                   |
| `.terminal !exit`      | 终止 VM 进程                           |
| `.terminal !run s`     | 在新进程中运行另一个 XCX 文件          |
| `.terminal !raw`       | 原始模式——无回显、无缓冲               |
| `.terminal !normal`    | 恢复普通终端模式                       |
| `.terminal !cursor on` | 显示光标                               |
| `.terminal !cursor off`| 隐藏光标                               |
| `.terminal !move x y`  | 将光标移动到位置 `x`、`y`（`i`）       |
| `.terminal !write expr`| 打印 `expr`，不追加换行                |

### 示例——游戏循环

```xcx
.terminal !raw;
.terminal !cursor off;
.terminal !clear;

while (true) do;
    s: k = input.key();
    if (k == "ESC") then; break; end;
    if (k == "UP") then; y = y - 1; end;
    .terminal !move x y;
    .terminal !write "@";
    @wait 16;
end;

.terminal !cursor on;
.terminal !normal;
```

## 错误处理与约束

- **普通模式下的 `input.key()`**：返回 `""` 并打印警告 alert。
- **对 `input.ready()` 使用 `@wait`**：编译错误。
- **`.terminal` 指令**：不是表达式，不能赋值给变量。
- **`.terminal !move`**：参数必须为整数（`i`）。
- **终端可用性**：若控制台句柄被重定向或不可用，VM 将以 fatal 错误终止。
