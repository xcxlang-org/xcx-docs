# XCX 3.1 Podstawy składni

## Komentarze

```xcx
--- Single-line comment
i: price = 100;  --- inline comment

---
Multi-line
comment block.
*---
```

## Identyfikatory

Identyfikatory są **wrażliwe na wielkość liter** i muszą pasować do wzorca `[a-zA-Z][a-zA-Z0-9_]*`.

| Prawidłowy    | Nieprawidłowy | Powód                              |
|---------------|---------------|------------------------------------|
| `userData`    | `1stUser`     | Nie może zaczynać się od cyfry      |
| `user_data`   | `user-data`   | Myślnik jest operatorem minus      |
| `counter1`    | `user name`   | Spacje nie są dozwolone            |

Słowa kluczowe (`if`, `func`, `fiber`, `i`, `f`, `s`, `b` itd.) są zarezerwowane.

## Końce instrukcji

Instrukcje kończą się średnikiem `;`. Dotyczy to deklaracji zmiennych, wywołań funkcji, przypisań i dyrektyw.

```xcx
i: x = 10;
>! "Hello";
```

## Bloki

Bloki otwierane są słowem kluczowym (np. `then`, `do`, `{`) i zamykane przez `end;`.

```xcx
if (x > 0) then;
    >! "Positive";
end;
```
