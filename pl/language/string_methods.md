# XCX 3.1 Metody łańcuchów znaków

Obiekty łańcuchów znaków w XCX są niemutowalne. Metody zwracają **nowy łańcuch** i nie modyfikują oryginału.

## Właściwości

- `.length`: Zwraca liczbę punktów kodowych Unicode w łańcuchu (np. `"zażółć".length` to 6). **Używana bez nawiasów.**

## Sekwencje escape

Literały łańcuchów znaków obsługują standardowe sekwencje escape:

| Sekwencja | Efekt              |
|-----------|--------------------|
| `\n`      | Nowa linia         |
| `\t`      | Tabulator poziomy  |
| `\r`      | Powrót karetki     |
| `\"`      | Cudzysłów          |
| `\\`      | Backslash          |
| `\xNN`    | Znak szesnastkowy (np. `\x1b`) |
| `\NNN`    | Znak ósemkowy (np. `\033`)     |

## Metody

| Metoda               | Sygnatura    | Opis                                              |
|----------------------|--------------|---------------------------------------------------|
| `.upper()`           | `() → s`     | Konwertuje wszystkie znaki na wielkie litery.     |
| `.lower()`           | `() → s`     | Konwertuje wszystkie znaki na małe litery.        |
| `.trim()`            | `() → s`     | Usuwa białe znaki z początku i końca.             |
| `.replace(f, t)`     | `(s, s) → s` | Zamienia wszystkie wystąpienia `f` na `t`.        |
| `.slice(s, e)`       | `(i, i) → s` | Zwraca podłańcuch od indeksu `s` do `e`.          |
| `.indexOf(s)`        | `(s) → i`    | Zwraca indeks pierwszego wystąpienia lub -1.      |
| `.lastIndexOf(s)`    | `(s) → i`    | Zwraca indeks ostatniego wystąpienia lub -1.      |
| `.startsWith(s)`     | `(s) → b`    | Zwraca true, jeśli łańcuch zaczyna się od `s`.    |
| `.endsWith(s)`       | `(s) → b`    | Zwraca true, jeśli łańcuch kończy się na `s`.     |
| `.toInt()`           | `() → i`     | Parsuje łańcuch na liczbę całkowitą; `halt.error` przy błędzie. |
| `.toFloat()`         | `() → f`     | Parsuje łańcuch na liczbę zmiennoprzecinkową; `halt.error` przy błędzie. |
| `.split(s)`          | `(s) → array:s` | Dzieli według separatora `s`; zwraca tablicę łańcuchów. |

## Przykłady

```xcx
s: raw = "  XCX-Language  ";
s: clean = raw.trim().lower().replace("-", "_"); --- "xcx_language"

i: start = "Programming".indexOf("gram");       --- 3
s: part = "Programming".slice(0, 4);            --- "Prog"

i: age = "25".toInt();
array:s: parts = "a,b,c".split(",");           --- {"a", "b", "c"}
```
