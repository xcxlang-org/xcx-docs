# XCX 3.1 Zmienne i stałe

## Deklaracje zmiennych

Zmienne deklaruje się składnią `type: name = value;`.

```xcx
i: age = 25;
f: pi = 3.14159;
s: name = "XCX";
b: is_ready = true;
```

Zmienne można również deklarować bez wartości początkowej — wówczas przyjmują wartość domyślną dla swojego typu (np. `0` dla `i`, `""` dla `s`).

## Ponowne przypisanie

Po deklaracji zmienne można ponownie przypisywać za pomocą operatora `=`.

```xcx
i: count = 0;
count = count + 1;
```

## Stałe

Stałe deklaruje się słowem kluczowym `const`. Muszą być zainicjalizowane przy deklaracji i nie można ich później zmieniać.

```xcx
const i: MAX_CONNECTIONS = 1024;
const s: VERSION = "2.0.0";
```

## Brak shadowingu zmiennych

XCX **nie** obsługuje shadowingu zmiennych. Deklaracja zmiennej o tej samej nazwie w dowolnym zakresie — w tym w zagnieżdżonym bloku — jest **błędem kompilacji** (`RedefinedVariable`).

```xcx
i: x = 10;
if (true) then;
    i: x = 20;   --- COMPILE ERROR: RedefinedVariable
end;
```

Jeśli trzeba zmienić wartość wewnątrz bloku, użyj ponownego przypisania zamiast ponownej deklaracji:

```xcx
i: x = 10;
if (true) then;
    x = 20;      --- OK: reassignment to existing variable
    >! x;        --- 20
end;
>! x;            --- 20 (the global variable was modified)
```
