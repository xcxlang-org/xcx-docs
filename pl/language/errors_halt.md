# XCX 3.1 Obsługa błędów (Halt)

XCX używa ustrukturyzowanego systemu `halt` do zarządzania warunkami wykonania i błędami w czasie działania.

## Poziomy halt

| Poziom        | Zachowanie                                              | Zastosowanie                      |
|---------------|---------------------------------------------------------|-----------------------------------|
| `halt.alert`  | Wypisuje komunikat; wykonanie trwa dalej.               | Logowanie, niekrytyczne ostrzeżenia |
| `halt.error`  | Wypisuje na stderr; przerywa bieżącą ramkę.             | Błędy logiczne z możliwością odzyskania |
| `halt.fatal`  | Wypisuje komunikat; natychmiast kończy maszynę wirtualną. | Krytyczna awaria, naruszenie bezpieczeństwa |

## Przykłady

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

## Paniki semantyczne i w czasie wykonania

Niektóre nieprawidłowe operacje skutkują automatyczną **paniką** (równoważną `halt.fatal` lub `halt.error` w zależności od kontekstu):
- **Dzielenie przez zero** (arytmetyka)
- **Błąd parsowania JSON**: Nieprawidłowy łańcuch w `json.parse()` powoduje natychmiastowe zakończenie maszyny wirtualnej.
- **Path Traversal**: Użycie `..` w metodach `store` lub ścieżek absolutnych poza katalogiem głównym projektu wywołuje `halt.fatal`.
- **Głębokość rekurencji**: Przekroczenie 800 ramek wywołuje `halt.error`.
