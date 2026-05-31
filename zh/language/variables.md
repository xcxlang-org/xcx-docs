# XCX 3.1 变量与常量

## 变量声明

变量使用 `type: name = value;` 语法声明。

```xcx
i: age = 25;
f: pi = 3.14159;
s: name = "XCX";
b: is_ready = true;
```

变量也可以不赋初值声明，此时将使用该类型的默认值（例如 `i` 为 `0`，`s` 为 `""`）。

## 重新赋值

声明后，变量可使用 `=` 运算符重新赋值。

```xcx
i: count = 0;
count = count + 1;
```

## 常量

常量使用 `const` 关键字声明。必须在声明时初始化，且之后不可更改。

```xcx
const i: MAX_CONNECTIONS = 1024;
const s: VERSION = "2.0.0";
```

## 不支持变量遮蔽

XCX **不支持**变量遮蔽。在任何作用域（包括嵌套块）中声明同名变量均为**编译时错误**（`RedefinedVariable`）。

```xcx
i: x = 10;
if (true) then;
    i: x = 20;   --- COMPILE ERROR: RedefinedVariable
end;
```

若需在块内修改变量值，应使用重新赋值而非重新声明：

```xcx
i: x = 10;
if (true) then;
    x = 20;      --- OK: reassignment to existing variable
    >! x;        --- 20
end;
>! x;            --- 20 (the global variable was modified)
```
