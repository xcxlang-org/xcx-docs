# XCX 3.1 I/O i polecenia systemowe

## Wyjście konsolowe (`>!`)

Operator `>!` wypisuje wartości na `stdout`, po czym dodaje znak nowej linii.

```xcx
>! "Hello";
>! 42;
>! "Path: " + path;
```

Obsługiwane są **sekwencje escape**, takie jak `\n` (nowa linia), `\t` (tabulator) i `\r` (powrót karetki).

## Wejście użytkownika (`>?`)

Operator `>?` odczytuje linię z `stdin` i próbuje sparsować ją do docelowej zmiennej.

```xcx
i: age;
>! "Enter age:";
>? age;
```

## Opóźnienie (`@wait`)

Wstrzymuje wykonanie maszyny wirtualnej na określoną liczbę milisekund.

```xcx
@wait 1000; --- Waits for 1 second
```

> `@wait` to operacja **blokująca**. Nie oddaje sterowania włóknem.

## Surowe wejście (`input`)

Moduł `input` umożliwia odczyt pojedynczych klawiszy bez oczekiwania na Enter.

### Metody

| Wywołanie           | Zwraca | Opis                                                    |
|---------------------|--------|---------------------------------------------------------|
| `input.key()`       | `s`    | Zwraca klawisz, jeśli dostępny; `""` w przeciwnym razie |
| `input.key() @wait` | `s`    | Czeka na klawisz i go zwraca                            |
| `input.ready()`     | `b`    | `true`, jeśli klawisz czeka w buforze                   |

### Stałe klawiszy

| Wartość         | Klawisz         |
|-----------------|-----------------|
| `"UP"`          | Strzałka w górę |
| `"DOWN"`        | Strzałka w dół  |
| `"LEFT"`        | Strzałka w lewo |
| `"RIGHT"`       | Strzałka w prawo |
| `"ENTER"`       | Enter           |
| `"ESC"`         | Escape          |
| `"BACKSPACE"`   | Backspace       |
| `"TAB"`         | Tab             |
| `"F1"` ... `"F12"` | F1–F12       |
| `"KEY_CTRL_C"`  | Ctrl+C          |
| `"KEY_CTRL_Z"`  | Ctrl+Z          |
| `"KEY_CTRL_S"`  | Ctrl+S          |

Zwykłe znaki zwracane są bezpośrednio: `"a"`, `"Z"`, `"5"`, `" "`.

### Przykład

```xcx
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

## Polecenia terminala (`.terminal`)

Bezpośrednia interakcja ze środowiskiem systemowym lub bieżącym procesem.

| Dyrektywa              | Opis                                              |
|------------------------|---------------------------------------------------|
| `.terminal !clear`     | Czyści ekran                                      |
| `.terminal !exit`      | Kończy proces maszyny wirtualnej                  |
| `.terminal !run s`     | Uruchamia inny plik XCX w nowym procesie          |
| `.terminal !raw`       | Tryb raw — bez echa, bez buforowania              |
| `.terminal !normal`    | Przywraca normalny tryb terminala                 |
| `.terminal !cursor on` | Pokazuje kursor                                 |
| `.terminal !cursor off`| Ukrywa kursor                                     |
| `.terminal !move x y`  | Przesuwa kursor na pozycję `x`, `y` (`i`)         |
| `.terminal !write expr`| Wypisuje `expr` bez końcowej nowej linii          |

### Przykład — pętla gry

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

## Obsługa błędów i ograniczenia

- **`input.key()` w trybie normalnym**: Zwraca `""` i wypisuje alert ostrzegawczy.
- **`@wait` na `input.ready()`**: Błąd kompilacji.
- **Dyrektywy `.terminal`**: Nie są wyrażeniami i nie można ich przypisać do zmiennych.
- **`.terminal !move`**: Argumenty muszą być liczbami całkowitymi (`i`).
- **Dostępność terminala**: Maszyna wirtualna zakończy działanie błędem fatal, jeśli uchwyty konsoli są przekierowane lub niedostępne.
