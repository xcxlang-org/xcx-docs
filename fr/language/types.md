# Types de Données XCX 3.1

## Types Simples

| Symbole       | Type         | Défaut       | Exemple                    |
|---------------|-------------|--------------|----------------------------|
| `i` / `int`   | Entier      | `0`          | `42`, `-7`, `0`            |
| `f` / `float` | Flottant    | `0.0`        | `3.14`, `-0.5`, `2.0`      |
| `s` / `str`   | Chaîne      | `""`         | `"hello"`, `""`            |
| `b` / `bool`  | Booléen     | `false`      | `true`, `false`            |
| `date`        | Date        | `1970-01-01` | `date("2024-12-25")`       |
| `json`        | Objet JSON  | `null`       | `<<< {"key": "value"} >>>` |

## Types Complexes

| Symbole      | Type                       | Exemple de Déclaration                       |
|--------------|---------------------------|----------------------------------------------|
| `array:i`    | Tableau d'entiers         | `array:i: nums {1, 2, 3}`                    |
| `array:f`    | Tableau de flottants      | `array:f: vals {1.0, 2.5}`                   |
| `array:s`    | Tableau de chaînes        | `array:s: words {"a", "b"}`                  |
| `array:b`    | Tableau de booléens       | `array:b: flags {true, false}`               |
| `array:json` | Tableau d'objets JSON     | `array:json: items`                          |
| `set:N`      | Ensemble de naturels      | `set:N: s {1,,10}`                           |
| `set:Z`      | Ensemble d'entiers        | `set:Z: s {-2, 0, 2}`                        |
| `set:Q`      | Ensemble de rationnels    | `set:Q: s {0.5, 1.0, 1.5}`                   |
| `set:S`      | Ensemble de chaînes       | `set:S: s {"a", "b"}`                        |
| `set:B`      | Ensemble de booléens      | `set:B: s {true, false}`                     |
| `set:C`      | Ensemble de caractères    | `set:C: s {"A",,"Z"}`                        |
| `map`        | Map clé-valeur            | `map: m { schema = [s <-> i] data = [...] }` |
| `table`      | Table relationnelle       | `table: t { columns = [...] rows = [...] }`  |
| `database:`  | Connexion à la base       | `database: db { engine = "sqlite", path = "app.db" }` |
| `fiber:T`    | Fibre typée               | `fiber:b: f = my_fiber(arg)`                 |
| `fiber:`     | Fibre void                | `fiber: f = my_void_fiber(arg)`              |

### array:json

`array:json` est un tableau stockant des éléments de type `json`. Utilisé principalement pour traiter des réponses HTTP renvoyant des tableaux d'objets JSON.

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

Les méthodes sur `array:json` sont identiques aux autres types de tableaux (`.push()`, `.pop()`, `.get(i)`, `.size()`, etc.).

### database:

`database:` représente une connexion à une base de données relationnelle. Déclarée avec un bloc de configuration et utilisée via ses méthodes intégrées. Voir la [Documentation Base de Données](database.md) pour l'API complète.

```xcx
database: app {
    engine = "sqlite",
    path   = "app.db"
};

yield app.sync(users);
table: all = yield app.fetch(users);
```

## Valeurs par Défaut

```xcx
int:   def_int;     --- 0 (alias for i)
float: def_float;   --- 0.0 (alias for f)
str:   def_str;     --- "" (alias for s)
bool:  def_bool;    --- false (alias for b)
```

## Conversion de Types

```xcx
f: x = 3.7;
i: n = i(x);       --- 3 (truncate toward zero)

i: m = 42;
f: y = f(m);       --- 42.0

i: num = 99;
s: str = s(num);   --- "99"
```

> [!NOTE]
> La conversion `b → i` est **intentionnellement bloquée**. `true + 1` est une erreur de type pour éviter les bugs logiques courants dans les langages de la famille C.