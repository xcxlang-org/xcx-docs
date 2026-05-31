# XCX 3.1 控制流

## 条件语句（if/elseif/else）

```xcx
if (condition) then;
    --- block
elseif (other_condition) then;
    --- block
else;
    --- block
end;
```

**别名：**

| 关键字   | 别名           |
|----------|----------------|
| `elseif` | `elif`, `elf`  |
| `else`   | `els`          |

所有形式等价，可混合使用：

```xcx
i: score = 75;
if (score >= 90) then;
    >! "A";
elif (score >= 75) then;
    >! "B";
elf (score >= 60) then;
    >! "C";
els;
    >! "F";
end;
```

---

## 循环

### 范围 for

范围在**两端均包含**边界值。

```xcx
for i in 1 to 3 do;
    >! i;
end;
--- prints: 1, 2, 3
```

### 步进 for（`@step`）

```xcx
for j in 0 to 6 @step 2 do;
    >! j;
end;
--- prints: 0, 2, 4, 6
```

### 集合迭代

适用于数组、集合和纤程。循环变量接收的是**元素**，而非索引。

```xcx
--- Array
array:i: nums {10, 20, 30};
for el in nums do;
    >! el;
end;

--- Set
set:N: primes {2, 3, 5, 7, 11};
for p in primes do;
    >! p;
end;

--- Fiber
fiber:i: f = gen(5);
for val in f do;
    >! val;
end;
```

### while 循环

```xcx
i: cnt = 0;
while (cnt < 3) do;
    cnt = cnt + 1;
    >! cnt;
end;
```

### break 与 continue

`break` 退出当前循环。`continue` 跳至下一次迭代。两者仅影响**直接包裹的循环**。

```xcx
for n in 1 to 5 do;
    if (n % 2 == 0) then; continue; end;
    if (n == 5) then; break; end;
    >! n;
end;
```

> [!NOTE]
> 在纤程的 `for` 循环内使用 `break` 时，会自动对该纤程调用 `.close()`。
