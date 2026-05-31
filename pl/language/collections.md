# XCX 3.1 Kolekcje

## Tablice

```xcx
array:i: nums {10, 20, 30};
nums.size();           --- 3
nums.get(0);           --- 10
nums.push(40);         --- adds 40 to the end
i: last = nums.pop();  --- removes and returns last element
nums.sort();           --- sorts in-place
nums.reverse();        --- reverses in-place
nums.show();           --- prints contents to terminal
```

### Metody tablic

| Metoda            | Sygnatura    | Zwraca | Opis                                                                 |
|-------------------|--------------|--------|----------------------------------------------------------------------|
| `.size()`         | `() → i`     | `i`    | Liczba elementów                                                     |
| `.get(i)`         | `(i) → T`    | `T`    | Element na pozycji `i` (indeksowanie od 0); `halt.error` poza zakresem |
| `.push(val)`      | `(T) → b`    | `b`    | Dodaje element na końcu                                              |
| `.pop()`          | `() → T`     | `T`     | Usuwa i zwraca ostatni element                                       |
| `.insert(i, val)` | `(i, T) → b` | `b`    | Wstawia na pozycji `i`, przesuwa resztę; `halt.error` poza zakresem  |
| `.update(i, val)` | `(i, T) → b` | `b`    | Nadpisuje element na pozycji `i`; `halt.error` poza zakresem         |
| `.delete(i)`      | `(i) → b`    | `b`    | Usuwa element na pozycji `i`; `halt.error` poza zakresem             |
| `.find(val)`      | `(T) → i`    | `i`    | Indeks pierwszego wystąpienia lub `-1`                               |
| `.contains(val)`  | `(T) → b`    | `b`    | Sprawdza, czy wartość istnieje                                       |
| `.isEmpty()`      | `() → b`     | `b`    | `true`, jeśli pusta                                                  |
| `.clear()`        | `() → b`     | `b`    | Usuwa wszystkie elementy                                             |
| `.sort()`         | `() → b`     | `b`    | Sortuje rosnąco (w miejscu)                                          |
| `.reverse()`      | `() → b`     | `b`    | Odwraca kolejność (w miejscu)                                        |
| `.toStr()`        | `() → s`     | `s`    | Serializuje tablicę do ciągu w formacie JSON                         |
| `.toJson()`       | `() → json`  | `json` | Konwertuje tablicę do natywnej struktury JSON                        |
| `.show()`         | `() → b`     | `b`    | Wypisuje zawartość w terminalu                                       |

```xcx
array:i: nums {5, 2, 8, 1};
nums.sort();            --- {1, 2, 5, 8}
nums.reverse();         --- {8, 5, 2, 1}
nums.push(99);          --- {8, 5, 2, 1, 99}
i: last = nums.pop();   --- last = 99, nums = {8, 5, 2, 1}
nums.insert(1, 15);     --- inserts 15 at position 1
nums.update(0, 5);      --- sets element 0 to 5
nums.delete(3);         --- removes element at position 3
b: found = nums.contains(5);
i: idx   = nums.find(5);
b: empty = nums.isEmpty();
```

---

## Zbiory

### Domeny

| Symbol | Typ              | Przykład                       |
|--------|------------------|--------------------------------|
| `N`    | Naturalne (≥ 0)  | `set:N: s {0, 1, 2}`           |
| `Z`    | Całkowite        | `set:Z: s {-3, 0, 3}`          |
| `Q`    | Wymierne (Float) | `set:Q: s {0.5, 1.0}`          |
| `S`    | Ciąg znaków      | `set:S: s {"a", "b"}`          |
| `B`    | Logiczny         | `set:B: s {true, false}`       |
| `C`    | Znak             | `set:C: s {"A",,"Z"}`          |

### Inicjalizacja

Zbiory można inicjalizować jawnymi wartościami lub zakresami. Zakresy są **włączne** z obu stron.

```xcx
set:N: small  {1,,5};                  --- {1, 2, 3, 4, 5}
set:N: evens  {0,,100 @step 2};        --- {0, 2, 4, ...}
set:Q: thirds {0.0,,1.0 @step 0.33};
set:C: letters {"A",,"Z"};            --- all uppercase letters
```

Zbiory **automatycznie usuwają duplikaty** elementów.

### Operacje na zbiorach

```xcx
set:N: setA {1,,5};
set:N: setB {3,,7};

set:N: u  = setA UNION setB;
set:N: i  = setA INTERSECTION setB;
set:N: d  = setA DIFFERENCE setB;
set:N: sd = setA SYMMETRIC_DIFFERENCE setB;

--- Unicode symbols are equivalent
setA ∪ setB
setA ∩ setB
setA \ setB
setA ⊕ setB
```

### Metody zbiorów

| Metoda          | Sygnatura | Zwraca | Opis                                           |
|-----------------|-----------|--------|------------------------------------------------|
| `.size()`       | `() → i`  | `i`    | Liczba elementów                               |
| `.isEmpty()`    | `() → b`  | `b`    | `true`, jeśli pusty                            |
| `.contains(v)`  | `(T) → b` | `b`    | Sprawdza przynależność                         |
| `.add(v)`       | `(T) → b` | `b`    | Dodaje element (ignoruje duplikat)             |
| `.remove(v)`    | `(T) → b` | `b`    | Usuwa element (brak efektu, jeśli nie ma)       |
| `.clear()`      | `() → b`  | `b`    | Usuwa wszystkie elementy                       |
| `.show()`       | `() → b`  | `b`    | Wypisuje `{elem, elem, ...}` w terminalu       |

### Losowy wybór i iteracja

```xcx
--- Random selection from a set:
i: picked_set = random.choice from small;

--- Random selection from an array:
array:i: nums {1, 2, 3, 4, 5};
i: picked_arr = random.choice from nums;
```

Wybór z pustego zbioru lub tablicy zwraca `false`.

### Iteracja

```xcx
for p in small do;
    >! p;
end;
```

---

## Mapy

```xcx
map: ages {
    schema = [s <-> i]
    data = [ "alice" :: 30, "bob" :: 25 ]
};

--- Empty Map
map: scores {
    schema = [s <-> i]   --- both separators are equivalent (<-> and <=>)
    data = [EMPTY]
};
```

### Metody map

| Metoda           | Sygnatura       | Zwraca    | Opis                                              |
|------------------|-----------------|-----------|---------------------------------------------------|
| `.size()`        | `() → i`        | `i`       | Liczba par klucz–wartość                          |
| `.get(key)`      | `(K) → V`       | `V`       | Zwraca wartość; `halt.error`, jeśli brak klucza   |
| `.contains(key)` | `(K) → b`       | `b`       | Sprawdza, czy klucz istnieje                      |
| `.insert(k, v)`  | `(K, V) → b`    | `b`       | Wstawia lub nadpisuje                             |
| `.remove(key)`   | `(K) → b`       | `b`       | Usuwa parę; `false`, jeśli brak klucza            |
| `.keys()`        | `() → array:K`  | `array:K` | Zwraca tablicę kluczy                             |
| `.values()`      | `() → array:V`  | `array:V` | Zwraca tablicę wartości                           |
| `.clear()`       | `() → b`        | `b`       | Usuwa wszystkie pary                              |
| `.toStr()`       | `() → s`        | `s`       | Serializuje mapę do ciągu w formacie JSON           |
| `.show()`        | `() → b`        | `b`       | Wypisuje zawartość mapy w terminalu               |
| `.toJson()`      | `() → json`     | `json`    | Serializuje mapę do obiektu JSON                  |

Klucze mapy są konwertowane na ciągi znaków w wynikowym obiekcie JSON.

### Serializacja mapy (toJson)

#### Sygnatura
`.toJson() → json`

#### Opis
Serializuje mapę do obiektu JSON. Wszystkie klucze są konwertowane na ciągi znaków za pomocą reprezentacji `.toString()`, aby spełnić wymagania dotyczące kluczy obiektów JSON.

```xcx
map: scores {
    schema = [s <-> i]
    data = [ "alice" :: 100, "bob" :: 85 ]
};
json: j = scores.toJson();
```

**Wynik JSON:**
```json
{
    "alice": 100,
    "bob": 85
}
```

#### Zachowanie w przypadkach szczególnych
- **Pusta mapa**: Zwraca pusty obiekt `{}`.
- **Struktury zagnieżdżone**: Jeśli mapa zawiera tablice, inne mapy lub tabele, są one rekurencyjnie serializowane do odpowiedników JSON.
- **Konwersja kluczy**: Wszystkie klucze mapy są konwertowane na ciągi znaków w wynikowym obiekcie JSON za pomocą standardowej reprezentacji `.toString()`.

#### Mapowanie typów
| Typ XCX | Typ JSON |
|---------|----------|
| `i`     | `number` |
| `f`     | `number` |
| `s`     | `string` |
| `b`     | `boolean` |
| `date`  | `string` (format `"YYYY-MM-DD HH:mm:ss"`) |
| `json`  | (bez zmian) |

Zawsze używaj `.contains()` przed `.get()`:

```xcx
if (ages.contains("alice")) then;
    >! ages.get("alice");
end;
```

---

## Tabele

Relacyjne struktury danych z opcjonalnymi kolumnami autoinkrementowanymi.

```xcx
table: products {
    columns = [ id :: i @auto, name :: s, price :: f ]
    rows = [ ("Laptop", 2999.99), ("Phone", 1499.50) ]
};

--- Empty Table
table: logs {
    columns = [ id :: i @auto, msg :: s ]
    rows = [EMPTY]
};
```

Modyfikator `@auto` na kolumnie typu `i` tworzy autoinkrementowane ID — jest pomijany w `.insert()` i `.add()`.

> [!NOTE]
> Dodatkowe atrybuty kolumn (`@pk`, `@unique`, `@optional`, `@default(v)`, `@fk(t.col)`) są używane przy łączeniu tabeli z bazą danych. Szczegóły w [Dokumentacji bazy danych](database.md).

### Dostęp do wierszy

```xcx
products[0].name    --- "Laptop" (sugar for .get(0))
products[1].price   --- 1499.50
```

### Metody tabel

| Metoda               | Sygnatura               | Zwraca | Opis                                                    |
|----------------------|-------------------------|--------|---------------------------------------------------------|
| `.count()`           | `() → i`                | `i`    | Liczba wierszy                                          |
| `.get(i)`            | `(i) → row`             | `row`  | Wiersz o indeksie `i`                                   |
| `.insert(vals...)`   | `(T...) → b`            | `b`    | Dodaje wiersz (pomija kolumny `@auto`)                  |
| `.add(vals...)`      | `(T...) → b`            | `b`    | Alias dla `.insert()` — identyczne zachowanie           |
| `.update(i, vals)`   | `(i, [T...]) → b`       | `b`    | Zastępuje wartości wiersza; kolumny `@auto` zachowane   |
| `.delete(i)`         | `(i) → b`               | `b`    | Usuwa wiersz o indeksie `i`                             |
| `.where(pred)`       | `(expr) → table`        | `table`| Filtruje — zwraca nową tabelę                           |
| `.join(t, pred)`     | `(table, pred) → table` | `table`| Wewnętrzne złączenie z inną tabelą                      |
| `.toJson()`          | `() → json`             | `json` | Serializuje wszystkie wiersze do tablicy JSON obiektów  |
| `.show()`            | `() → b`                | `b`    | Wypisuje tabelę w formacie ASCII                        |

### Argumenty nazwane dla `.add()` i `.insert()`

Gdy tabela ma atrybuty kolumn bazy danych, wartości można przekazywać według nazwy kolumny zamiast pozycji. Argumenty nazwane są **opcjonalne** — wywołania pozycyjne pozostają w pełni poprawne.

```xcx
table: users {
    columns = [
        id    :: i @auto @pk,
        name  :: s @unique,
        age   :: i,
        phone :: s @optional,
        role  :: s @default("user")
    ]
    rows = [EMPTY]
};

--- Positional (backward compatible)
users.add("Alice", 25, "", "user");

--- Named
users.add(name = "Alice", age = 25, phone = "", role = "user");

--- Mixed — positional args must come first
users.add("Alice", age = 25, role = "admin");
```

**Rozdzielenie przestrzeni nazw.** Lewa strona `=` to zawsze nazwa kolumny. Prawa strona to wyrażenie z lokalnego zakresu. To dwie niezależne przestrzenie nazw — bez konfliktu:

```xcx
s: name = "Alice";
users.add(name = name, age = 25);
--- left "name"  = column users.name
--- right "name" = local variable
```

Reguły: argumenty pozycyjne muszą poprzedzać nazwane; kolumn `@auto` nigdy nie można przekazać; pominięcie wymaganej kolumny (bez `@optional`, bez `@default`) to błąd kompilacji; zduplikowane nazwy kolumn w tym samym wywołaniu to błąd kompilacji. Pełna specyfikacja w [Dokumentacji bazy danych](database.md).

### Filtrowanie (where)

```xcx
--- Shorthand syntax (column names usable directly)
table: expensive = products.where(price > 1000.0);
table: named     = products.where(name HAS "Pro");

--- Lambda
table: r = products.where(row -> row.price > 1000.0);

--- Chaining
table: result = products
    .where(price > 1000.0)
    .where(name HAS "Pro");
```

> [!IMPORTANT]
> **Konflikty nazw w `.where()` (S301)**: Nazwy kolumn mają pierwszeństwo przed zmiennymi lokalnymi w predykatach. Jeśli zmienna lokalna ma taką samą nazwę jak kolumna, zmień nazwę zmiennej, aby uniknąć błędu kompilacji.
>
> ```xcx
> --- Wrong (conflict: 'token' exists both as column and parameter)
> fiber verify(s: token) {
>     table: sess = db.sessions.where(token == token);
> };
>
> --- Correct
> fiber verify(s: t) {
>     table: sess = db.sessions.where(token == t);
> };
> ```

### Złączenia

```xcx
--- Key-based join
table: report = users.join(orders, "id", "user_id");

--- Lambda join
table: custom = tableA.join(tableB, (a, b) -> a.id == b.ref_id);
```

Gdy połączone tabele mają wspólną nazwę kolumny (inną niż klucz złączenia), wynikowa kolumna jest prefiksowana `{table_name}_`.

### Serializacja (toJson)

#### Sygnatura
`.toJson() → json`

#### Opis
Serializuje wszystkie wiersze tabeli do tablicy JSON, gdzie każdy wiersz staje się obiektem z kluczami odpowiadającymi nazwom kolumn. Kolumny `@auto` są uwzględnione w wyniku.

#### Format
Zawsze zwraca tablicę JSON (`[...]`). Pusta tabela zwraca `[]`.

```xcx
table: products {
    columns = [ id :: i @auto, name :: s, price :: f ]
    rows = [ ("Laptop", 2999.99), ("Phone", 1499.50) ]
};

json: result = products.toJson();
```

**Wynik JSON:**
```json
[
    {"id": 1, "name": "Laptop", "price": 2999.99},
    {"id": 2, "name": "Phone",  "price": 1499.50}
]
```

#### Mapowanie typów
| Typ XCX         | Typ JSON                                  |
|-----------------|-------------------------------------------|
| `i` / `int`     | `number`                                  |
| `f` / `float`   | `number`                                  |
| `s` / `str`     | `string`                                  |
| `b` / `bool`    | `boolean`                                 |
| `date`          | `string` (format `"YYYY-MM-DD HH:mm:ss"`) |

#### Zachowanie w przypadkach szczególnych
- **Pusta tabela**: Zwraca pustą tablicę `[]`.
- **Przefiltrowana tabela**: Serializowane są tylko wiersze obecne w tabeli (po `.where()`).
- **Połączona tabela**: Uwzględniane są wszystkie kolumny z wyniku złączenia.
