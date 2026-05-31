# XCX 3.1 制御フロー

## 条件分岐（if/elseif/else）

```xcx
if (condition) then;
    --- block
elseif (other_condition) then;
    --- block
else;
    --- block
end;
```

**別名：**

| キーワード | 別名           |
|-----------|----------------|
| `elseif`  | `elif`, `elf`  |
| `else`    | `els`          |

すべての形式は等価であり、混在して使用できます：

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

## ループ

### 範囲 for

範囲は両端とも**包含的**です。

```xcx
for i in 1 to 3 do;
    >! i;
end;
--- prints: 1, 2, 3
```

### ステップ付き for（`@step`）

```xcx
for j in 0 to 6 @step 2 do;
    >! j;
end;
--- prints: 0, 2, 4, 6
```

### コレクションの反復

配列、集合、ファイバで使用できます。ループ変数には**インデックスではなく要素**が代入されます。

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

### while ループ

```xcx
i: cnt = 0;
while (cnt < 3) do;
    cnt = cnt + 1;
    >! cnt;
end;
```

### break と continue

`break` は現在のループを抜けます。`continue` は次の反復に進みます。どちらも**直近の外側のループ**にのみ影響します。

```xcx
for n in 1 to 5 do;
    if (n % 2 == 0) then; continue; end;
    if (n == 5) then; break; end;
    >! n;
end;
```

> [!NOTE]
> ファイバに対する `for` ループ内で `break` すると、そのファイバに対して自動的に `.close()` が呼ばれます。
