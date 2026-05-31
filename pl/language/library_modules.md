# XCX 3.1 Biblioteka standardowa i moduły

## Wbudowane moduły

### crypto

Narzędzia kryptograficzne.

| Metoda                              | Zwraca | Opis                                           |
|-------------------------------------|--------|------------------------------------------------|
| `crypto.hash(data, "bcrypt")`       | `s`    | Haszuje hasło algorytmem bcrypt                |
| `crypto.hash(data, "argon2")`       | `s`    | Haszuje hasło algorytmem argon2 (zalecane)     |
| `crypto.hash(data, "base64_encode")`| `s`    | Koduje dane binarne/łańcuch do Base64          |
| `crypto.hash(data, "base64_decode")`| `s`    | Dekoduje łańcuch Base64 z powrotem do binarnego/łańcucha |
| `crypto.verify(password, hash, algo)` | `b`  | Zwraca `true`, jeśli hasło pasuje do hasha     |
| `crypto.token(length)`              | `s`    | Generuje losowy token hex o podanej długości   |

```xcx
s: hash  = crypto.hash(password, "bcrypt");
s: hash2 = crypto.hash(password, "argon2");
b: valid = crypto.verify(password, hash2, "argon2");
s: token = crypto.token(32);
```

### store (File I/O)

Wszystkie ścieżki muszą być **względne** względem katalogu głównego projektu. Ścieżki absolutne lub path traversal (`..`) wywołują `halt.fatal`.

| Metoda              | Sygnatura                 | Zwraca | Opis                                           |
|---------------------|---------------------------|--------|------------------------------------------------|
| `store.write(p, c)` | `(s, s) → b`              | `b`    | Nadpisuje plik. Tworzy katalogi w razie potrzeby. |
| `store.read(p)`     | `(s) → s`                 | `s`    | Zwraca zawartość pliku; `halt.fatal` jeśli brak. |
| `store.append(p, c)`| `(s, s) → b`              | `b`    | Dopisuje do pliku. Tworzy, jeśli brak.         |
| `store.exists(p)`   | `(s) → b`                 | `b`    | Sprawdza istnienie. Bez efektów ubocznych.     |
| `store.delete(p)`   | `(s) → b`                 | `b`    | Usuwa plik lub katalog (rekurencyjnie).        |
| `store.list(p)`     | `(s) → array:s`           | `array:s`| Zwraca listę plików i folderów.              |
| `store.isDir(p)`    | `(s) → b`                 | `b`    | `true`, jeśli ścieżka to katalog.              |
| `store.size(p)`     | `(s) → i`                 | `i`    | Rozmiar pliku w bajtach.                       |
| `store.mkdir(p)`    | `(s) → b`                 | `b`    | Tworzy katalog (rekurencyjnie).                 |
| `store.glob(pat)`   | `(s) → array:s`           | `array:s`| Zwraca pliki pasujące do wzorca glob.        |
| `store.zip(s, t)`   | `(s, s) → b`              | `b`    | Archiwizuje źródło do docelowego zip.          |
| `store.unzip(z, d)` | `(s, s) → b`              | `b`    | Rozpakowuje zip do miejsca docelowego.         |

```xcx
store.write("log.txt", "First line");
store.append("log.txt", "\nSecond line");
s: content = store.read("log.txt");
if (store.exists("lock.pid")) then;
    >! "Already running";
end;
```

### env

| Metoda          | Sygnatura      | Zwraca    | Opis                                                      |
|-----------------|----------------|-----------|-----------------------------------------------------------|
| `env.get(name)` | `(s) → s`      | `s`       | Zwraca wartość zmiennej środowiskowej; `halt.error` jeśli nie ustawiona |
| `env.args()`    | `() → array:s` | `array:s` | Zwraca argumenty CLI przekazane programowi jako tablicę |

```xcx
s: db_url = env.get("DATABASE_URL");

array:s: args = env.args();
for arg in args do;
    >! arg;
end;
```

### random

| Metoda                            | Sygnatura           | Zwraca | Opis                                                              |
|-----------------------------------|---------------------|--------|-------------------------------------------------------------------|
| `random.choice from col`          | `(set/array) → T`   | `T`    | Losuje element z podanego zbioru lub tablicy.                     |
| `random.int(min, max @step num)`  | `(i, i, @i) → i`    | `i`    | Losuje liczbę całkowitą z zakresu `[min, max]`. Domyślny krok: `1`. |
| `random.float(min, max @step num)`| `(f, f, @f) → f`    | `f`    | Losuje liczbę zmiennoprzecinkową z zakresu `[min, max]`. Domyślny krok: `0.5`. |

```xcx
set:N: pool {1,,10};
i: picked = random.choice from pool;

--- Works on arrays too
array:s: words {"hello", "world"};
s: w = random.choice from words;

--- Picks 1, 3, 5, 7, or 9
i: odd = random.int(1, 10 @step 2);

--- Picks 0.0, 0.5, 1.0, 1.5, or 2.0
f: weight = random.float(0.0, 2.0);

--- Picks 0.0, 0.25, 0.5, 0.75, 1.0
f: precision = random.float(0.0, 1.0 @step 0.25);
```

### date (module)

```xcx
date: now = date.now();
```

---

## System modułów

### include

Scal kod z innego pliku z bieżącą przestrzenią nazw.

```xcx
include "utils.xcx";
include "math.xcx" as m;

m.PI;
m.sqrt(16.0);
```

Bez aliasu — wszystkie symbole są dostępne bezpośrednio w bieżącej przestrzeni nazw. Z aliasem — wszystkie symbole najwyższego poziomu mają prefiks: `alias.symbol`.

Zależności cykliczne są wykrywane i odrzucane w czasie kompilacji.
