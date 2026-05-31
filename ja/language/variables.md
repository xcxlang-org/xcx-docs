# XCX 3.1 変数と定数

## 変数宣言

変数は `type: name = value;` 構文で宣言します。

```xcx
i: age = 25;
f: pi = 3.14159;
s: name = "XCX";
b: is_ready = true;
```

初期値なしでも変数を宣言できます。この場合、型のデフォルト値が使われます（例：`i` は `0`、`s` は `""`）。

## 再代入

宣言後、変数は `=` 演算子で再代入できます。

```xcx
i: count = 0;
count = count + 1;
```

## 定数

定数は `const` キーワードで宣言します。宣言時に初期化が必須で、以降は変更できません。

```xcx
const i: MAX_CONNECTIONS = 1024;
const s: VERSION = "2.0.0";
```

## 変数のシャドウイング禁止

XCX は変数のシャドウイングを**サポートしません**。入れ子ブロックを含む任意のスコープで同じ名前の変数を宣言すると、**コンパイル時エラー**（`RedefinedVariable`）になります。

```xcx
i: x = 10;
if (true) then;
    i: x = 20;   --- COMPILE ERROR: RedefinedVariable
end;
```

ブロック内で値を変更する必要がある場合は、再宣言ではなく再代入を使用してください：

```xcx
i: x = 10;
if (true) then;
    x = 20;      --- OK: reassignment to existing variable
    >! x;        --- 20
end;
>! x;            --- 20 (the global variable was modified)
```
