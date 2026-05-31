# XCX 3.1 I/O とシステムコマンド

## コンソール出力（`>!`）

`>!` 演算子は値を `stdout` に出力し、続けて改行します。

```xcx
>! "Hello";
>! 42;
>! "Path: " + path;
```

**エスケープシーケンス** `\n`（改行）、`\t`（タブ）、`\r`（キャリッジリターン）などがサポートされています。

## ユーザー入力（`>?`）

`>?` 演算子は `stdin` から 1 行を読み取り、対象変数へのパースを試みます。

```xcx
i: age;
>! "Enter age:";
>? age;
```

## 遅延（`@wait`）

VM の実行を指定ミリ秒だけ一時停止します。

```xcx
@wait 1000; --- Waits for 1 second
```

> `@wait` は **ブロッキング** 操作です。ファイバーを yield しません。

## 生入力（`input`）

`input` モジュールは、Enter を待たずに 1 キーを読み取れます。

### メソッド

| 呼び出し                | 戻り値 | 説明                                             |
|---------------------|---------|---------------------------------------------------------|
| `input.key()`       | `s`     | キーがあれば返す。なければ `""`                   |
| `input.key() @wait` | `s`     | キー入力を待って返す                          |
| `input.ready()`     | `b`     | バッファにキーがあれば `true`                |

### キー定数

| 値           | キー             |
|-----------------|-----------------|
| `"UP"`          | 上矢印        |
| `"DOWN"`        | 下矢印      |
| `"LEFT"`        | 左矢印      |
| `"RIGHT"`       | 右矢印       |
| `"ENTER"`       | Enter           |
| `"ESC"`         | Escape          |
| `"BACKSPACE"`   | Backspace     |
| `"TAB"`         | Tab             |
| `"F1"` ... `"F12"` | F1–F12 |
| `"KEY_CTRL_C"`  | Ctrl+C          |
| `"KEY_CTRL_Z"`  | Ctrl+Z          |
| `"KEY_CTRL_S"`  | Ctrl+S          |

通常の文字はそのまま返されます: `"a"`、`"Z"`、`"5"`、`" "`。

### 例

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

## ターミナルコマンド（`.terminal`）

システム環境または現在のプロセスと直接やり取りします。

| ディレクティブ              | 説明                                      |
|------------------------|--------------------------------------------------|
| `.terminal !clear`     | 画面をクリア                                |
| `.terminal !exit`      | VM プロセスを終了                        |
| `.terminal !run s`     | 別プロセスで別の XCX ファイルを実行           |
| `.terminal !raw`       | 生モード — エコーなし、バッファリングなし                 |
| `.terminal !normal`    | 通常のターミナルモードに戻す                    |
| `.terminal !cursor on` | カーソルを表示                                 |
| `.terminal !cursor off`| カーソルを非表示                                 |
| `.terminal !move x y`  | カーソルを位置 `x`、`y`（`i`）へ移動      |
| `.terminal !write expr`| `expr` を末尾改行なしで出力         |

### 例 — ゲームループ

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

## エラー処理と制約

- **通常モードでの `input.key()`**: `""` を返し、警告アラートを表示します。
- **`input.ready()` への `@wait`**: コンパイルエラー。
- **`.terminal` ディレクティブ**: 式ではなく、変数に代入できません。
- **`.terminal !move`**: 引数は整数（`i`）でなければなりません。
- **ターミナルの利用可否**: コンソールハンドルがリダイレクトされている、または利用できない場合、VM は致命的エラーで停止します。
