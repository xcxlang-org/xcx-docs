# XCX 3.1 Typy danych

## Typy proste

| Symbol        | Typ              | Domyślna     | Przykład                    |
|---------------|------------------|--------------|-----------------------------|
| `i` / `int`   | Liczba całkowita | `0`          | `42`, `-7`, `0`             |
| `f` / `float` | Liczba zmiennoprzecinkowa | `0.0` | `3.14`, `-0.5`, `2.0` |
| `s` / `str`   | Łańcuch znaków   | `""`         | `"hello"`, `""`             |
| `b` / `bool`  | Wartość logiczna | `false`      | `true`, `false`             |
| `date`        | Data             | `1970-01-01` | `date("2024-12-25")`        |
| `json`        | Obiekt JSON      | `null`       | `<<< {"key": "value"} >>>`  |

## Typy złożone

| Symbol       | Typ                          | Przykład deklaracji                          |
|--------------|------------------------------|----------------------------------------------|
| `array:i`    | Tablica liczb całkowitych    | `array:i: nums {1, 2, 3}`                    |
| `array:f`    | Tablica liczb zmiennoprzecinkowych | `array:f: vals {1.0, 2.5}`             |
| `array:s`    | Tablica łańcuchów znaków     | `array:s: words {"a", "b"}`                  |
| `array:b`    | Tablica wartości logicznych  | `array:b: flags {true, false}`               |
| `array:json` | Tablica obiektów JSON        | `array:json: items`                          |
| `set:N`      | Zbiór liczb naturalnych      | `set:N: s {1,,10}`                           |
| `set:Z`      | Zbiór liczb całkowitych       | `set:Z: s {-2, 0, 2}`                        |
| `set:Q`      | Zbiór liczb wymiernych (float) | `set:Q: s {0.5, 1.0, 1.5}`                 |
| `set:S`      | Zbiór łańcuchów znaków        | `set:S: s {"a", "b"}`                        |
| `set:B`      | Zbiór wartości logicznych    | `set:B: s {true, false}`                     |
| `set:C`      | Zbiór znaków                 | `set:C: s {"A",,"Z"}`                        |
| `map`        | Mapa klucz–wartość           | `map: m { schema = [s <-> i] data = [...] }` |
| `table`      | Tabela relacyjna             | `table: t { columns = [...] rows = [...] }`  |
| `database:`  | Połączenie z bazą danych     | `database: db { engine = "sqlite", path = "app.db" }` |
| `fiber:T`    | Włókno typowane              | `fiber:b: f = my_fiber(arg)`                 |
| `fiber:`     | Włókno void                  | `fiber: f = my_void_fiber(arg)`              |

### array:json

`array:json` to tablica przechowująca elementy typu `json`. Używana głównie przy pracy z odpowiedziami HTTP zwracającymi tablice obiektów JSON.

```xcx
--- Declare an empty JSON array
array:json: items;

--- Iterate over a JSON array (e.g. from a network response)
i: size = resp.body.size();
i: idx = 0;
while (idx < size) do;
    json: item = resp.body.get(idx);
    s: name; item.bind("name", name);
    >! name;
    idx = idx + 1;
end;

--- Add an element
json: obj <<< {"key": "value"} >>>;
items.push(obj);
```

Metody `array:json` są identyczne jak w innych typach tablicowych (`.push()`, `.pop()`, `.get(i)`, `.size()` itd.).

### database:

`database:` reprezentuje połączenie z relacyjną bazą danych. Deklaruje się je blokiem konfiguracyjnym i używa przez wbudowane metody. Pełne API opisuje [Dokumentacja bazy danych](database.md).

```xcx
database: app {
    engine = "sqlite",
    path   = "app.db"
};

yield app.sync(users);
table: all = yield app.fetch(users);
```

## Wartości domyślne

```xcx
int:   def_int;     --- 0 (alias for i)
float: def_float;   --- 0.0 (alias for f)
str:   def_str;     --- "" (alias for s)
bool:  def_bool;    --- false (alias for b)
```

## Rzutowanie typów

```xcx
f: x = 3.7;
i: n = i(x);       --- 3 (truncate toward zero)

i: m = 42;
f: y = f(m);       --- 42.0

i: num = 99;
s: str = s(num);   --- "99"
```

> [!NOTE]
> Konwersja `b → i` jest **celowo zablokowana**. Wyrażenie `true + 1` jest błędem typu, aby zapobiec logicznym błędom typowym dla języków z rodziny C.
