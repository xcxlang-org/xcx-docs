# XCX 3.1 语法基础

## 注释

```xcx
--- Single-line comment
i: price = 100;  --- inline comment

---
Multi-line
comment block.
*---
```

## 标识符

标识符**区分大小写**，且必须匹配模式 `[a-zA-Z][a-zA-Z0-9_]*`。

| 有效          | 无效         | 原因                     |
|---------------|-------------|--------------------------|
| `userData`    | `1stUser`    | 不能以数字开头           |
| `user_data`   | `user-data`  | 连字符是减号运算符       |
| `counter1`    | `user name`  | 不允许空格               |

关键字（`if`、`func`、`fiber`、`i`、`f`、`s`、`b` 等）为保留字。

## 语句终止符

语句以分号 `;` 结尾。适用于变量声明、函数调用、赋值和指令。

```xcx
i: x = 10;
>! "Hello";
```

## 代码块

代码块由关键字（如 `then`、`do`、`{`）开启，由 `end;` 关闭。

```xcx
if (x > 0) then;
    >! "Positive";
end;
```
