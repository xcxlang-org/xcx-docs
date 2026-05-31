# XCX 3.1 — дата и время

## Создание

Даты создаются функцией `date()` или `date.now()`.

```xcx
date: d1 = date("2024-12-25");               --- YYYY-MM-DD
date: d2 = date("25/12/2024", "DD/MM/YYYY"); --- Custom format
date: now = date.now();                      --- Current system time
```

## Свойства

Объекты даты предоставляют целочисленные поля только для чтения:

- `.year`
- `.month` (1-12)
- `.day` (1-31)
- `.hour` (0-23)
- `.minute` (0-59)
- `.second` (0-59)

## Арифметика и сравнение

С датами можно складывать и вычитать дни, а также сравнивать их друг с другом.

```xcx
date: tomorrow = some_date + 1;
date: yesterday = some_date - 1;
i: days_between = christmas - halloween;  --- Result in integer days
b: is_before = christmas < new_year;
```

## Форматирование

```xcx
s: s1 = now.format();                    --- Default: "YYYY-MM-DD HH:mm:ss"
s: s2 = now.format("DD/MM/YYYY HH:mm");  --- Custom output
```

**Токены:** `YYYY` (год), `MM` (месяц 01–12), `DD` (день 01–31), `HH` (час 00–23), `mm` (минута), `ss` (секунда), `SSS` или `ms` (миллисекунды), `M` и `D` (без ведущего нуля).
