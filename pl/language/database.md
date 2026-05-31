# XCX 3.1 Baza danych

XCX 3.1 wprowadza natywne wsparcie dla relacyjnych baz danych przez typ `database:` i zestaw wbudowanych metod. Wersja 3.1 obsługuje wyłącznie **SQLite**.

---

## Spis treści

1. [Deklaracja połączenia (`database:`)](#1-deklaracja-połączenia-database)
2. [Atrybuty kolumn tabeli](#2-atrybuty-kolumn-tabeli)
3. [Argumenty nazwane](#3-argumenty-nazwane)
4. [Przechwytywanie wyniku (`as`)](#4-przechwytywanie-wyniku-as)
5. [Ograniczenia zakresu włókien (Windows)](#5-ograniczenia-zakresu-włókien-windows)
6. [Referencja API](#6-referencja-api)
   - 5.1 [DDL](#51-ddl)
   - 5.2 [Operacje zapisu](#52-operacje-zapisu)
   - 5.3 [Operacje odczytu](#53-operacje-odczytu)
   - 5.4 [Operacje usuwania](#54-operacje-usuwania)
   - 5.5 [Transakcje](#55-transakcje)
   - 5.6 [Inne](#56-inne)
6. [Mapowanie typów (XCX ↔ SQL)](#6-mapowanie-typów-xcx--sql)
7. [Ograniczenia bezpieczeństwa](#7-ograniczenia-bezpieczeństwa)
8. [Pełny przykład](#8-pełny-przykład)

---

## 1. Deklaracja połączenia (`database:`)

```xcx
database: app {
    engine  = "sqlite",
    path    = "app.db"
};
```

### Pola

| Pole       | Typ | Wymagane | Domyślne | Opis                                                      |
|------------|-----|----------|----------|-----------------------------------------------------------|
| `engine`   | `s` | tak      | —        | Silnik bazy danych. Obecnie tylko `"sqlite"`.             |
| `path`     | `s` | tak      | —        | Ścieżka do pliku `.db` (względem katalogu głównego projektu). |
| `timeout`  | `i` | nie      | `5000`   | Limit czasu operacji w ms.                                |
| `readonly` | `b` | nie      | `false`  | Tryb tylko do odczytu.                                    |

### Zachowanie

- Połączenia są **leniwe** — otwierane przy pierwszym użyciu.
- Przy nieudanym połączeniu (np. brak uprawnień) automatycznie wywoływany jest `halt.error`.
- SQLite tworzy plik bazy danych, jeśli nie istnieje (chyba że `readonly = true`).
- Dozwolone są wielokrotne równoczesne połączenia — każde ma własną nazwę.

```xcx
database: main { engine = "sqlite", path = "main.db" };
database: logs { engine = "sqlite", path = "logs.db" };

yield main.sync(users);
yield logs.sync(events);
```

### Model I/O

Wszystkie operacje DB dotykające dysku lub sterownika to operacje I/O:

| Kontekst        | Zachowanie              |
|-----------------|-------------------------|
| Wewnątrz włókna | wymaga `yield`          |
| Poza włóknem    | blokuje synchronicznie  |

Dotyczy to operacji **odczytu** (`fetch`, `query`, `queryRaw`), **zapisu** (`push`, `save`, `insert`, `exec`), **usuwania** (`remove`, `exec`) i **DDL** (`sync`, `drop`).

**Wyjątek — metody bez `yield`.** Metody operujące wyłącznie na stanie transakcji lub metadanych nie wymagają `yield` w żadnym kontekście:

- `db.has()`, `db.close()`, `db.isOpen()`
- `db.begin()`, `db.commit()`, `db.rollback()`

> `begin()`, `commit()` i `rollback()` nie dotykają danych — zarządzają tylko stanem transakcji w sterowniku.

---

## 2. Atrybuty kolumn tabeli

Te atrybuty rozszerzają deklarację bloku `table:` i są używane przez moduł bazy danych.

### Atrybuty

| Atrybut       | Opis                                                                 | Odpowiednik SQL     |
|---------------|----------------------------------------------------------------------|---------------------|
| `@pk`         | Klucz główny. Można łączyć z `@auto`.                                | `PRIMARY KEY`       |
| `@unique`     | Wartość musi być unikalna w kolumnie.                                | `UNIQUE`            |
| `@optional`   | Kolumna może być `NULL` w SQL. XCX zwraca wartość domyślną typu przy odczycie. | `NULL` |
| `@default(v)` | Domyślna wartość kolumny w SQL.                                      | `DEFAULT v`         |
| `@fk(t.col)`  | Klucz obcy wskazujący kolumnę `col` tabeli `t`.                      | `REFERENCES t(col)` |

### Przykład

```xcx
table: users {
    columns = [
        id      :: i @auto @pk,
        name    :: s @unique,
        age     :: i,
        phone   :: s @optional,
        role    :: s @default("user")
    ]
    rows = [EMPTY]
};

table: posts {
    columns = [
        id      :: i @auto @pk,
        user_id :: i @fk(users.id),
        title   :: s,
        body    :: s @optional
    ]
    rows = [EMPTY]
};
```

### Zachowanie `@optional`

XCX nie ma `null` jako samodzielnego typu. Kolumny `@optional` w SQL mogą przechowywać `NULL`, ale przy odczycie do XCX wartość jest automatycznie zastępowana domyślną dla typu:

| Typ XCX | Wartość, gdy SQL NULL |
|---------|------------------------|
| `i`     | `0`                    |
| `f`     | `0.0`                  |
| `s`     | `""`                   |
| `b`     | `false`                |

> **Uwaga:** `@optional` sygnalizuje tylko akceptowalność `NULL` po stronie SQL. XCX nie zachowuje rozróżnienia między `NULL` a wartością domyślną — ta informacja ginie przy odczycie. Jeśli logika aplikacji wymaga rozróżnienia tych przypadków, przechowuj flagę w osobnej kolumnie.

### Reguły pomijania kolumn dla `add()` i `insert()`

| Atrybut kolumny          | Można pominąć? | Co trafia do SQL              |
|--------------------------|----------------|-------------------------------|
| `@auto`                  | tak (zawsze)   | wartość auto-generowana       |
| `@default(v)`            | tak            | wartość `v` z `DEFAULT`       |
| `@optional`              | tak            | `NULL`                        |
| `@optional @default(v)`  | tak            | wartość `v` z `DEFAULT`       |
| brak atrybutów           | **nie**        | błąd kompilacji               |

> Kolumn `@auto` nigdy nie można podać jawnie — ani pozycyjnie, ani przez argument nazwany. Próba to **błąd kompilacji**.

---

## 3. Argumenty nazwane

Argumenty nazwane pozwalają przekazywać wartości do `table.add()`, `table.insert()` i `db.insert()` **według nazwy kolumny** zamiast pozycji. To opcjonalna, addytywna funkcja — wszystkie istniejące wywołania pozycyjne pozostają w pełni poprawne.

Argumenty nazwane działają **wyłącznie** dla operacji wstawiania do tabel. Nie dotyczą funkcji użytkownika, włókien ani innych wbudowanych metod. Nazwy kolumn są statycznie znane kompilatorowi ze schematu `table:`.

### Składnia

```xcx
--- Positional (backward compatible)
users.add("Alice", 25, "", "user");

--- Named
users.add(name = "Alice", age = 25, phone = "", role = "user");

--- With db.insert()
yield app.insert(users, name = "Alice", age = 25) as saved;
```

**Rozdzielenie przestrzeni nazw.** Lewa strona `=` to zawsze nazwa kolumny. Prawa strona to wyrażenie z lokalnego zakresu. To dwie niezależne przestrzenie nazw — bez konfliktu:

```xcx
s: name = "Alice";
users.add(name = name, age = 25);
--- left "name"  = column users.name
--- right "name" = local variable
```

**Kolejność ewaluacji.** Wyrażenia po prawej stronie są ewaluowane od lewej do prawej w kolejności zapisu:

```xcx
users.add(name = get_name(), age = get_age());
--- get_name() is called before get_age()
```

### Mieszanie pozycyjnych i nazwanych

Argumenty pozycyjne i nazwane można mieszać. **Argumenty pozycyjne muszą poprzedzać nazwane.**

```xcx
--- OK — positional before named
users.add("Alice", age = 25, role = "admin");

--- COMPILE ERROR — named before positional
users.add(name = "Alice", 25, "admin");
```

**Reguła przypisania.** Argumenty pozycyjne są przypisywane do kolumn od lewej do prawej w kolejności deklaracji, **pomijając `@auto`**, aż się wyczerpią. Argumenty nazwane wypełniają pozostałe kolumny według nazwy.

```xcx
--- "Alice" → name, rest named
users.add("Alice", age = 25, role = "admin");
--- phone omitted → NULL (@optional)

--- "Alice", 25 → name, age; role named, phone omitted
users.add("Alice", 25, role = "mod");
```

Argumenty pozycyjne przypisują sekwencyjnie **bez pomijania**. Pominięcie środkowej kolumny jest możliwe tylko przez argument nazwany lub całkowite pominięcie (jeśli opcjonalna):

```xcx
--- COMPILE ERROR — compiler assigns: name="Alice", age="" — type mismatch
--- there is no way to "skip" age positionally
users.add("Alice", "", "user");
```

Podanie tej samej kolumny pozycyjnie i nazwą to **błąd kompilacji**:

```xcx
--- COMPILE ERROR — name provided twice
users.add("Alice", name = "Bob", age = 25);
```

### Reguły kompilacji

| Reguła                              | Zachowanie        |
|-------------------------------------|-------------------|
| Podano kolumnę `@auto`              | błąd kompilacji   |
| Nieznana nazwa kolumny              | błąd kompilacji   |
| Zduplikowana nazwa kolumny          | błąd kompilacji   |
| Pominięto wymaganą kolumnę          | błąd kompilacji   |
| Argument nazwany przed pozycyjnym   | błąd kompilacji   |

**Tabela kompletności:**

| Atrybut kolumny          | Można pominąć? | Wartość przy pominięciu              |
|--------------------------|----------------|--------------------------------------|
| `@auto`                  | tak (zawsze)   | wartość auto-generowana              |
| `@default(v)`            | tak            | wartość `v` z `DEFAULT`              |
| `@optional`              | tak            | `NULL` (→ domyślna XCX przy odczycie) |
| `@optional @default(v)`  | tak            | wartość `v` z `DEFAULT`              |
| brak atrybutów           | **nie**        | błąd kompilacji                      |

```xcx
--- OK — role has @default("user"), phone is @optional
users.add(name = "Alice", age = 25);

--- COMPILE ERROR — age is required
users.add(name = "Alice");
```

---

## 4. Przechwytywanie wyniku (`as`)

`as` to ogólny mechanizm XCX do przechwytywania wyniku operacji blokowej:

```xcx
--- HTTP request
net.request { method = "GET", url = "..." } as resp;

--- DB operations
yield app.insert(users, name = "Alice", age = 25) as saved;
yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
```

### Obiekt wyniku zapisu/usuwania

Operacje `db.insert()`, `db.save()`, `db.push()` i `db.exec()` zwracają obiekt z dwoma polami:

| Pole       | Typ | Opis                                                              |
|------------|-----|-------------------------------------------------------------------|
| `affected` | `i` | Liczba wierszy zmienionych / wstawionych przez operację           |
| `insertId` | `i` | ID ostatniego wstawionego rekordu (`@auto @pk`); `0`, jeśli brak wstawienia |

```xcx
yield app.insert(users, name = "Alice", age = 25) as saved;
>! "New user ID: " + s(saved.insertId);

yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
>! "Deleted: " + s(res.affected);
```

Użycie `as` jest opcjonalne — jeśli wynik nie jest potrzebny, można go pominąć:

```xcx
yield app.save(users);
```

### Semantyka `affected` i `insertId`

| Operacja                   | `affected`               | `insertId`                    |
|----------------------------|--------------------------|-------------------------------|
| `insert()` — jeden wiersz  | `1`                      | ID wstawionego wiersza        |
| `push()` — wiele wierszy   | liczba wstawionych wierszy | ID ostatniego wstawionego wiersza |
| `save()` — upsert (insert) | `1`                      | ID wstawionego wiersza        |
| `save()` — upsert (update) | `1`                      | `0`                           |
| `exec(DELETE ...)`         | liczba usuniętych wierszy | `0`                          |
| `exec(UPDATE ...)`         | liczba zaktualizowanych wierszy | `0`                    |

> Przy niepowodzeniu operacji (naruszenie ograniczenia, brak połączenia, `readonly`) wywoływany jest `halt.error` i wykonanie nie dociera do kodu używającego wyniku `as`. Obiekt wyniku jest zawsze poprawny, jeśli kod go widzi.

---

## 5. Ograniczenia zakresu włókien (Windows)

W XCX 3.1, szczególnie w środowiskach Windows, bezpośrednia inicjalizacja zmiennej lokalnej włókna wynikiem z bazy danych może sporadycznie prowadzić do błędu `Undefined variable` [S101] w kolejnych liniach.

### Zalecane obejście

Aby zapewnić pełną widoczność w blokach warunkowych wewnątrz włókna, zaleca się **najpierw zadeklarować typ zmiennej**, a następnie przypisać wynik bazy danych w osobnej instrukcji.

```xcx
fiber run(-> b) {
    --- Recommended pattern
    b: exists_before;
    exists_before = db.has(logs);
    
    if (exists_before) then;
        >! "Table exists";
    end;
}
```

Ten wzorzec zapewnia, że zmienna jest poprawnie zarejestrowana w tabeli symboli przed dostępem przez analizator. To ograniczenie ma zostać natywnie rozwiązane architektonicznie w XCX 4.0.

---

## 6. Referencja API

### 5.1 DDL

```xcx
--- Creates the SQL table if it does not exist (based on XCX table schema)
yield app.sync(users);

--- Drops the SQL table
yield app.drop(users);

--- Checks if table exists in the database (no data touched — no yield)
b: exists = app.has(users);
```

### 5.2 Operacje zapisu

```xcx
--- INSERT all rows from a local XCX table into SQL
yield app.push(users);

--- INSERT OR UPDATE (upsert) — requires @pk; no @pk = compile error
--- Implementation: INSERT INTO ... ON CONFLICT(pk) DO UPDATE SET ...
yield app.save(users);

--- INSERT a single row — positional or named
yield app.insert(users, "Alice", 25) as saved;
yield app.insert(users, name = "Alice", age = 25) as saved;
```

Przy błędzie (naruszenie `@unique`, `@fk`, brak połączenia) operacja wywołuje `halt.error`.

### 5.3 Operacje odczytu

```xcx
--- SELECT * — returns a native XCX table
table: all_users = yield app.fetch(users);

--- SELECT with XCX-style filter (compiles to SQL WHERE)
table: adults = yield app.fetch(users).where(age > 18);

--- SELECT with raw SQL — requires schema hint as first argument
table: result = yield app.query(users, "SELECT * FROM users WHERE age > ?", [18]);

--- Raw SQL without schema — always returns a JSON array
json: raw = yield app.queryRaw("SELECT COUNT(*) as n FROM users");
--- raw = [{"n": 42}]

--- Get the first row
json: row = yield app.queryRaw("SELECT COUNT(*) as n FROM users").first();
--- row = {"n": 42}
--- halt.error if result is empty
```

#### `.first()` na wyniku `queryRaw`

`.first()` zwraca pierwszy element tablicy JSON jako pojedynczy obiekt `json`. Wywołuje `halt.error`, jeśli zapytanie nie zwróciło wierszy. Typowe użycie to zapytania agregujące (`COUNT`, `SUM`, `MAX`) i wyszukiwania zwracające co najwyżej jeden wiersz.

```xcx
json: row = yield app.queryRaw("SELECT COUNT(*) as n FROM users").first();
i: total; row.bind("n", total);
>! "Users: " + s(total);
```

Jeśli nie masz pewności, że wynik będzie niepusty, sprawdź rozmiar przed użyciem `.first()`:

```xcx
json: results = yield app.queryRaw("SELECT * FROM users WHERE age > ?", [100]);
if (results.size() > 0) then;
    json: row = results.get(0);
end;
```

#### Obsługiwane operatory w `.where()`

Filtr `.where()` kompiluje się do klauzuli SQL `WHERE`. Obsługiwany jest ograniczony podzbiór wyrażeń XCX:

| Kategoria     | Obsługiwane                      |
|---------------|----------------------------------|
| Porównania    | `==`, `!=`, `<`, `<=`, `>`, `>=` |
| Logiczne      | `AND`, `OR`, `NOT`               |
| Wartości      | literały, zmienne lokalne        |
| Lewe operandy | tylko nazwy kolumn               |

Wywołania funkcji, metody ciągów i wyrażenia zagnieżdżone nie są obsługiwane w `.where()` z `db.fetch()`. Użycie nieobsługiwanego wyrażenia to **błąd kompilacji**.

```xcx
--- OK
table: r = yield app.fetch(users).where(age > 18 AND role == "admin");

--- COMPILE ERROR — string method not allowed in where() with fetch
table: r = yield app.fetch(users).where(name.lower() == "alice");
```

Łączenie `.where()` łączy warunki operatorem `AND`:

```xcx
table: r = yield app.fetch(users)
    .where(age > 18)
    .where(role == "admin");
--- SQL equivalent: WHERE age > 18 AND role = 'admin'
```

#### `db.fetch()` i lokalne `rows`

`db.fetch()` odczytuje tylko dane z bazy SQL — lokalne `rows` zadeklarowane w bloku `table:` są ignorowane. Lokalna definicja tabeli służy wyłącznie jako wskazówka schematu.

```xcx
table: users {
    columns = [ id :: i @auto @pk, name :: s ]
    rows = [ ("Alice") ]   --- these rows do NOT go into fetch
};

yield app.sync(users);

--- Returns only rows stored in app.db
table: all = yield app.fetch(users);
```

Aby wstawić lokalne `rows` do bazy danych, użyj `db.push(users)` przed `db.fetch()`.

### 5.4 Operacje usuwania

```xcx
--- DELETE with filter — .where() is required (D401)
yield app.remove(users).where(age < 18);

--- Delete all rows
yield app.truncate(users);

--- Raw DELETE with parameters
yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
```

> **`db.remove()` bez `.where()` to błąd kompilacji (D401).** Aby usunąć wszystkie wiersze, użyj jawnie `db.truncate(users)`.

### 5.5 Transakcje

```xcx
app.begin();
yield app.save(users);
yield app.save(posts);
app.commit();
```

> Jeśli jakakolwiek operacja w transakcji wywoła `halt.error`, transakcja jest **automatycznie wycofywana** przed przerwaniem ramki. Połączenie wraca do stanu sprzed `begin()`.

`rollback()` służy do jawnego anulowania transakcji w logice warunkowej przed wystąpieniem błędu:

```xcx
app.begin();
b: valid = validate_users(users);
if (NOT valid) then;
    app.rollback();
end;
yield app.save(users);
app.commit();
```

### 5.6 Inne

```xcx
app.close();
b: alive = app.isOpen();
```

---

## 6. Mapowanie typów (XCX ↔ SQL)

| Typ XCX | Typ SQL (SQLite)               |
|---------|--------------------------------|
| `i`     | `INTEGER`                      |
| `f`     | `REAL`                         |
| `s`     | `TEXT`                         |
| `b`     | `INTEGER` (0 / 1)              |
| `date`  | `TEXT` (`YYYY-MM-DD HH:mm:ss`) |

---

## 7. Ograniczenia bezpieczeństwa

| Ograniczenie                        | Zachowanie                                                 |
|-------------------------------------|------------------------------------------------------------|
| Ścieżka zawierająca `..`            | `halt.fatal` — traversing ścieżki                          |
| Ścieżka bezwzględna                 | `halt.fatal`                                               |
| Surowe SQL z niezwalidowanym wejściem | Deweloper odpowiada za sanityzację — używaj parametrów `?` |
| `readonly = true` + operacja DML    | `halt.error`                                               |
| `save()` na tabeli bez `@pk`        | błąd kompilacji                                            |
| `remove()` bez `.where()`           | błąd kompilacji (D401)                                     |

Zawsze używaj prepared statements z `?`:

```xcx
--- WRONG
app.exec("DELETE FROM users WHERE name = '" + name + "'");

--- CORRECT
app.exec("DELETE FROM users WHERE name = ?", [name]);
```

---

## 8. Pełny przykład

```xcx
database: app {
    engine = "sqlite",
    path   = "data.db"
};

table: users {
    columns = [
        id    :: i @auto @pk,
        name  :: s @unique,
        email :: s @unique,
        age   :: i,
        phone :: s @optional
    ]
    rows = [EMPTY]
};

yield app.sync(users);

fiber handle_get_users(json: req -> json) {
    table: all = yield app.fetch(users);
    yield net.respond(200, all.toJson());
};

fiber handle_create_user(json: req -> json) {
    json: body;
    req.bind("body", body);

    s: name;  body.bind("name", name);
    s: email; body.bind("email", email);
    i: age;   body.bind("age", age);

    --- Named arguments — readable and order-independent
    yield app.insert(users, name = name, email = email, age = age) as saved;

    json: resp <<< {"ok": true, "id": 0} >>>;
    resp.set("id", saved.insertId);
    yield net.respond(201, resp);
};

serve: api {
    port   = 8080,
    routes = [
        "GET  /users" :: handle_get_users,
        "POST /users" :: handle_create_user
    ]
};
```
