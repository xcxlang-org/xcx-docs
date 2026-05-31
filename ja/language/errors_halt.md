# XCX 3.1 エラー処理（Halt）

XCX はランタイムの状態とエラーを管理するため、構造化された `halt` システムを使います。

## Halt レベル

| レベル         | 挙動                                          | 用途                      |
|---------------|---------------------------------------------------|-------------------------------|
| `halt.alert`  | メッセージを表示。実行は継続。            | ログ、非致命的な警告|
| `halt.error`  | stderr に出力。現在のフレームを中止。       | 回復可能なロジックエラー      |
| `halt.fatal`  | メッセージを表示。VM を直ちに終了。 | 致命的な障害、セキュリティ違反 |

## 例

```xcx
halt.alert >! "Cache missed, fetching from DB...";

if (divisor == 0) then;
    halt.error >! "Division by zero!";
    return 0; --- Execution returns to caller from the recovery point
end;

if (NOT db.is_healthy()) then;
    halt.fatal >! "Database corrupted. Emergency shutdown.";
end;
```

## 意味論的・ランタイムのパニック

特定の無効な操作は、自動的に **パニック**（文脈により `halt.fatal` または `halt.error` に相当）を引き起こします。

- **ゼロ除算**（算術）
- **JSON パース失敗**: `json.parse()` に無効な文字列を渡すと、VM は直ちに終了します。
- **パストラバーサル**: `store` メソッドで `..` を使う、またはプロジェクトルート外の絶対パスは `halt.fatal` を発生させます。
- **再帰の深さ**: 800 フレームを超えると `halt.error` を発生させます。
