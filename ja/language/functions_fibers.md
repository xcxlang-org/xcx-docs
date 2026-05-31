# XCX 3.1 関数とファイバ

> [!IMPORTANT]
> **グローバルスコープのルール**：関数とファイバはファイルの**トップレベル**で宣言する必要があります。スコープに影響を与える形で `if`/`while`/`for` ブロックや他の関数/ファイバの内部にネストすることはできません — すべての関数名とファイバ名はコンパイル時にグローバルに登録されます。

## 関数

### 定義スタイル

パラメータは `type: name` 形式に従います（例：`i: count`）。

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

### 関数の呼び出し

```xcx
hello();
f: pi = get_pi();
greet("Alice");
i: result = add(3, 4);
```

### 再帰

最大深度：**800 フレーム**。これを超えると `halt.error` が発生します。

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

## ファイバ（コルーチン）

ファイバは `yield` で実行を中断し、再開できます。**型付き**（yield が値を返す）と **void**（値なしで yield）の 2 種類があります。

### 定義

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

### インスタンス化

```xcx
fiber:i: gen  = generator(10);   --- typed
fiber:  log   = logger("init");  --- void
```

### ファイバのメソッド

| メソッド    | 対象       | 戻り値 | 説明                                     |
|-------------|------------|--------|------------------------------------------|
| `.next()`   | `fiber:T`  | `T`    | 再開し、次に yield された値を返す        |
| `.run()`    | `fiber:`   | `b`    | void ファイバを 1 ステップ再開する       |
| `.isDone()` | 両方       | `b`    | ファイバが終了していれば `true`          |
| `.close()`  | 両方       | `b`    | ファイバを強制終了する                   |

`isDone() == true` の状態で `.next()` を呼ぶと `halt.alert`（R306）が発生します。

### yield

```xcx
--- In a typed fiber
yield true;
yield score + 10;

--- In a void fiber
yield;
```

### ファイバ内の return

1. **型付きファイバ**（`fiber ... -> T`）：型 `T` の値を伴う `return expr;` が必要です。値なしの `return;` はコンパイルエラー（S210）です。
2. **void ファイバ**：早期終了には `return;` を使用します。

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

### ファイバに対する for ループ

```xcx
fiber:i: f = generator(5);
for val in f do;
    >! val;
end;
```

`while (!f.isDone()) { val = f.next(); ... }` と等価です。ループ内の `break` は自動的に `.close()` を呼び出します。

### 関数パラメータとしてのファイバ

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

### yield from（委譲）

実行をサブファイバに委譲します。コンパイラはこれを、サブファイバからすべての値を yield するループに脱糖します。

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

### 手動委譲

```xcx
fiber pipeline(s: user, s: pass -> b) {
    fiber:b: step1 = check_user(user);
    yield step1.next();

    fiber:b: step2 = check_pass(user, pass);
    yield step2.next();

    return false;
};
```
