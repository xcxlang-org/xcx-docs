# XCX 3.1 Operatory

## Operatory arytmetyczne

| Operator | Opis                              | Przykład   | Wynik   |
|----------|-----------------------------------|------------|---------|
| `+`      | Dodawanie                         | `10 + 3`   | `13`    |
| `-`      | Odejmowanie / negacja jednoargumentowa | `10 - 3` | `7`     |
| `*`      | Mnożenie                          | `10 * 3`   | `30`    |
| `/`      | Dzielenie (obcina dla `i`)        | `10 / 3`   | `3`     |
| `%`      | Modulo (reszta)                   | `10 % 3`   | `1`     |
| `^`      | Potęgowanie                       | `2 ^ 10`   | `1024`  |
| `++`     | Konkatenacja cyfr całkowitych     | `48 ++ 77` | `4877`  |

## Operatory łańcuchów znaków

| Operator | Opis                    | Przykład                 | Wynik            |
|----------|-------------------------|--------------------------|------------------|
| `+`      | Konkatenacja            | `"Hi " + "User"`         | `"Hi User"`      |
| `HAS`    | Wyszukiwanie podłańcucha | `"Search" HAS "ear"`    | `true`           |

> [!TIP]
> Konkatenacja łańcuchów znaków automatycznie konwertuje liczby i wartości logiczne na łańcuchy (np. `"Value: " + 42` → `"Value: 42"`).

## Operatory logiczne

| Operator | Alias | Opis          |
|----------|-------|---------------|
| `AND`    | `&&`  | Logiczne AND  |
| `OR`     | `||`  | Logiczne OR   |
| `NOT`    | `!!`  | Logiczne NOT  |

## Operatory porównania

Służą do porównywania wartości tego samego typu.

| Operator | Opis                        |
|----------|-----------------------------|
| `==`     | Równe                       |
| `!=`     | Różne                       |
| `<`      | Mniejsze niż                |
| `>`      | Większe niż                 |
| `<=`     | Mniejsze lub równe          |
| `>=`     | Większe lub równe           |

## Operatory zbiorów

| Słowo                  | Symbol | Opis                      |
|------------------------|--------|---------------------------|
| `UNION`                | `∪`    | Suma zbiorów              |
| `INTERSECTION`         | `∩`    | Przecięcie zbiorów        |
| `DIFFERENCE`           | `\`    | Różnica zbiorów           |
| `SYMMETRIC_DIFFERENCE` | `⊕`    | Różnica symetryczna       |

## Kolejność operatorów (od najwyższego priorytetu)

1. Wywołania funkcji, wywołania metod, indeksowanie `[]`
2. Negacja jednoargumentowa `-`, `!!` (NOT)
3. Potęgowanie `^`
4. Mnożenie `*`, dzielenie `/`, modulo `%`
5. Operacje na zbiorach `UNION`, `INTERSECTION`, `DIFFERENCE`, `SYM_DIFF`
6. Dodawanie `+`, odejmowanie `-`, konkatenacja `++`
7. Porównania `HAS`, `>`, `<`, `>=`, `<=`
8. Równość `==`, `!=`
9. Logiczne `AND`, `&&`
10. Logiczne `OR`, `||`
