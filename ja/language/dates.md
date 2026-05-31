# XCX 3.1 日付と時刻

## 作成

日付は `date()` 関数または `date.now()` で作成します。

```xcx
date: d1 = date("2024-12-25");               --- YYYY-MM-DD
date: d2 = date("25/12/2024", "DD/MM/YYYY"); --- Custom format
date: now = date.now();                      --- Current system time
```

## プロパティ

日付オブジェクトは読み取り専用の整数フィールドを公開します。

- `.year`
- `.month`（1–12）
- `.day`（1–31）
- `.hour`（0–23）
- `.minute`（0–59）
- `.second`（0–59）

## 算術と比較

日付は日数の加減算と、他の日付との比較をサポートします。

```xcx
date: tomorrow = some_date + 1;
date: yesterday = some_date - 1;
i: days_between = christmas - halloween;  --- Result in integer days
b: is_before = christmas < new_year;
```

## 書式化

```xcx
s: s1 = now.format();                    --- Default: "YYYY-MM-DD HH:mm:ss"
s: s2 = now.format("DD/MM/YYYY HH:mm");  --- Custom output
```

**トークン:** `YYYY`（年）、`MM`（月 01–12）、`DD`（日 01–31）、`HH`（時 00–23）、`mm`（分）、`ss`（秒）、`SSS` または `ms`（ミリ秒）、`M` と `D`（先頭ゼロなし）。
