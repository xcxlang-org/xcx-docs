# XCX 3.1 日期与时间

## 创建

日期通过 `date()` 函数或 `date.now()` 创建。

```xcx
date: d1 = date("2024-12-25");               --- YYYY-MM-DD
date: d2 = date("25/12/2024", "DD/MM/YYYY"); --- Custom format
date: now = date.now();                      --- Current system time
```

## 属性

日期对象提供只读整型字段：

- `.year`
- `.month`（1–12）
- `.day`（1–31）
- `.hour`（0–23）
- `.minute`（0–59）
- `.second`（0–59）

## 算术与比较

日期支持加减天数，以及与其他日期比较。

```xcx
date: tomorrow = some_date + 1;
date: yesterday = some_date - 1;
i: days_between = christmas - halloween;  --- Result in integer days
b: is_before = christmas < new_year;
```

## 格式化

```xcx
s: s1 = now.format();                    --- Default: "YYYY-MM-DD HH:mm:ss"
s: s2 = now.format("DD/MM/YYYY HH:mm");  --- Custom output
```

**格式标记：** `YYYY`（年）、`MM`（月 01–12）、`DD`（日 01–31）、`HH`（时 00–23）、`mm`（分）、`ss`（秒）、`SSS` 或 `ms`（毫秒）、`M` 与 `D`（无前导零）。
