# XCX 3.1 函数与纤程

> [!IMPORTANT]
> **全局作用域规则**：函数和纤程必须在文件的**顶层**声明。不能嵌套在 `if`/`while`/`for` 块或其他函数/纤程内部（若会影响作用域）——所有函数名和纤程名在编译时全局注册。

## 函数

### 定义风格

参数遵循 `type: name` 模式（例如 `i: count`）。

```xcx
--- No parameters, no return value
func hello() {
    >! "Hello!";
};

--- No parameters, with return value
func get_pi(-> f) {
    return 3.14159;
};

--- With parameters, no return value
func greet(s: who) {
    >! "Hello, " + who + "!";
};

--- With parameters and return value
func add(i: x, i: y -> i) {
    return x + y;
};
```

### 调用函数

```xcx
hello();
f: pi = get_pi();
greet("Alice");
i: result = add(3, 4);
```

### 递归

最大深度：**800 帧**。超出此限制将触发 `halt.error`。

```xcx
func factorial(i: n -> i) {
    if (n <= 1) then;
        return 1;
    else;
        return n * factorial(n - 1);
    end;
};
>! factorial(5);   --- 120
```

---

## 纤程（协程）

纤程可通过 `yield` 暂停执行并在之后恢复。有两种变体：**带类型**（`yield` 返回值）和 **void**（`yield` 不返回值）。

### 定义

```xcx
--- Typed Fiber: each yield returns a value of type T
fiber generator(i: max -> i) {
    for i in 1 to max do;
        yield i;
    end;
    return 0;
};

--- Void Fiber: yield without a value
fiber logger(s: msg) {
    if (msg == "") then; return; end;
    >! msg;
    yield;
};
```

### 实例化

```xcx
fiber:i: gen  = generator(10);   --- typed
fiber:  log   = logger("init");  --- void
```

### 纤程方法

| 方法        | 适用对象   | 返回值 | 说明                              |
|-------------|------------|--------|-----------------------------------|
| `.next()`   | `fiber:T`  | `T`    | 恢复执行；返回下一个 yield 的值   |
| `.run()`    | `fiber:`   | `b`    | 恢复 void 纤程一步                |
| `.isDone()` | 两者       | `b`    | 纤程结束时为 `true`               |
| `.close()`  | 两者       | `b`    | 强制终止纤程                      |

在 `isDone() == true` 时调用 `.next()` 将触发 `halt.alert`（R306）。

### yield

```xcx
--- In a typed fiber
yield true;
yield score + 10;

--- In a void fiber
yield;
```

### 纤程中的 return

1. **带类型纤程**（`fiber ... -> T`）：需要 `return expr;`，且表达式类型为 `T`。无值的 `return;` 是编译错误（S210）。
2. **void 纤程**：使用 `return;` 提前退出。

```xcx
fiber:i: gen(i: max -> i) {
    for i in 1 to max do;
        if (i == 3) then;
            return 99;   --- Valid: terminates and returns 99
        end;
        yield i;
    end;
    return 0;            --- Required at end of typed fiber
};

fiber logger(s: msg) {
    if (msg == "") then;
        return;          --- Valid for void fiber
    end;
    >! msg;
};
```

### 遍历纤程的 for 循环

```xcx
fiber:i: f = generator(5);
for val in f do;
    >! val;
end;
```

等价于 `while (!f.isDone()) { val = f.next(); ... }`。循环内的 `break` 会自动调用 `.close()`。

### 纤程作为函数参数

```xcx
func run_checks(fiber:b: f) {
    while (f.isDone() == false) do;
        b: result = f.next();
        if (result == false) then;
            halt.alert >! "Step failed";
            return;
        end;
    end;
};

fiber:b: checks = login_pipeline(user, pass);
run_checks(checks);
```

### yield from（委托）

将执行委托给子纤程。编译器会将其脱糖为循环，依次 yield 子纤程的每个值。

```xcx
fiber count(i: n -> i) {
    i: x = 0;
    while (x < n) do;
        yield x;
        x = x + 1;
    end;
    return 0;
};

fiber delegator(-> i) {
    fiber:i: sub = count(3);
    yield from sub;   --- yields: 0, 1, 2
    yield 100;        --- then yields: 100
    return 0;
};
```

### 手动委托

```xcx
fiber pipeline(s: user, s: pass -> b) {
    fiber:b: step1 = check_user(user);
    yield step1.next();

    fiber:b: step2 = check_pass(user, pass);
    yield step2.next();

    return false;
};
```
