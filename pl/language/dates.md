# XCX 3.1 Data i czas

## Tworzenie

Daty tworzy się funkcją `date()` lub `date.now()`.

```xcx
date: d1 = date("2024-12-25");               --- YYYY-MM-DD
date: d2 = date("25/12/2024", "DD/MM/YYYY"); --- Custom format
date: now = date.now();                      --- Current system time
```

## Właściwości

Obiekty daty udostępniają tylko do odczytu pola całkowitoliczbowe:

- `.year`
- `.month` (1-12)
- `.day` (1-31)
- `.hour` (0-23)
- `.minute` (0-59)
- `.second` (0-59)

## Arytmetyka i porównania

Daty obsługują dodawanie/odejmowanie dni oraz porównywanie z innymi datami.

```xcx
date: tomorrow = some_date + 1;
date: yesterday = some_date - 1;
i: days_between = christmas - halloween;  --- Result in integer days
b: is_before = christmas < new_year;
```

## Formatowanie

```xcx
s: s1 = now.format();                    --- Default: "YYYY-MM-DD HH:mm:ss"
s: s2 = now.format("DD/MM/YYYY HH:mm");  --- Custom output
```

**Tokeny:** `YYYY` (rok), `MM` (miesiąc 01-12), `DD` (dzień 01-31), `HH` (godzina 00-23), `mm` (minuta), `ss` (sekunda), `SSS` lub `ms` (milisekundy), `M` i `D` (bez wiodącego zera).
