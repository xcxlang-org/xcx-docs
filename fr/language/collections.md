# Collections XCX 3.1

## Tableaux (Arrays)

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

### Méthodes de Tableau

| Méthode           | Signature    | Retour  | Description                                                          |
|-------------------|--------------|---------|----------------------------------------------------------------------|
| `.size()`         | `() → i`     | `i`     | Nombre d'éléments                                                    |
| `.get(i)`         | `(i) → T`    | `T`     | Élément à la position `i` (indexé à 0) ; `halt.error` si hors limites |
| `.push(val)`      | `(T) → b`    | `b`     | Ajoute un élément à la fin                                           |
| `.pop()`          | `() → T`     | `T`     | Retire et retourne le dernier élément                                |
| `.insert(i, val)` | `(i, T) → b` | `b`     | Insère à la position `i`, décale le reste ; `halt.error` si hors limites |
| `.update(i, val)` | `(i, T) → b` | `b`     | Écrase l'élément à la position `i` ; `halt.error` si hors limites   |
| `.delete(i)`      | `(i) → b`    | `b`     | Supprime l'élément à la position `i` ; `halt.error` si hors limites |
| `.find(val)`      | `(T) → i`    | `i`     | Index de la première occurrence, ou `-1`                             |
| `.contains(val)`  | `(T) → b`    | `b`     | Vérifie si la valeur existe                                          |
| `.isEmpty()`      | `() → b`     | `b`     | `true` si vide                                                       |
| `.clear()`        | `() → b`     | `b`     | Supprime tous les éléments                                           |
| `.sort()`         | `() → b`     | `b`     | Trie par ordre croissant (en place)                                  |
| `.reverse()`      | `() → b`     | `b`     | Inverse l'ordre (en place)                                           |
| `.toStr()`        | `() → s`     | `s`     | Sérialise le tableau en chaîne JSON                                  |
| `.toJson()`       | `() → json`  | `json`  | Convertit le tableau en structure JSON native                        |
| `.show()`         | `() → b`     | `b`     | Affiche le contenu dans le terminal                                  |

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

## Ensembles (Sets)

### Domaines

| Symbole | Type                 | Exemple                       |
|---------|----------------------|-------------------------------|
| `N`     | Naturels (≥ 0)       | `set:N: s {0, 1, 2}`          |
| `Z`     | Entiers              | `set:Z: s {-3, 0, 3}`         |
| `Q`     | Rationnels (Flottant)| `set:Q: s {0.5, 1.0}`         |
| `S`     | Chaînes              | `set:S: s {"a", "b"}`         |
| `B`     | Booléens             | `set:B: s {true, false}`      |
| `C`     | Caractères           | `set:C: s {"A",,"Z"}`         |

### Initialisation

Les ensembles peuvent être initialisés avec des valeurs explicites ou des plages. Les plages sont **inclusives** des deux côtés.

```xcx
set:N: small  {1,,5};                  --- {1, 2, 3, 4, 5}
set:N: evens  {0,,100 @step 2};        --- {0, 2, 4, ...}
set:Q: thirds {0.0,,1.0 @step 0.33};
set:C: letters {"A",,"Z"};            --- all uppercase letters
```

Les ensembles **dédupliquent automatiquement** les éléments.

### Opérations sur les Ensembles

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

### Méthodes d'Ensemble

| Méthode         | Signature | Retour | Description                                  |
|-----------------|-----------|--------|----------------------------------------------|
| `.size()`       | `() → i`  | `i`    | Nombre d'éléments                            |
| `.isEmpty()`    | `() → b`  | `b`    | `true` si vide                               |
| `.contains(v)`  | `(T) → b` | `b`    | Vérifie l'appartenance                       |
| `.add(v)`       | `(T) → b` | `b`    | Ajoute un élément (ignore les doublons)      |
| `.remove(v)`    | `(T) → b` | `b`    | Supprime un élément (no-op si absent)        |
| `.clear()`      | `() → b`  | `b`    | Supprime tous les éléments                   |
| `.show()`       | `() → b`  | `b`    | Affiche `{elem, elem, ...}` dans le terminal |

### Sélection Aléatoire et Itération

```xcx
--- Random selection from a set:
i: picked_set = random.choice from small;

--- Random selection from an array:
array:i: nums {1, 2, 3, 4, 5};
i: picked_arr = random.choice from nums;
```

Piocher dans un ensemble ou tableau vide retourne `false`.

### Itération

```xcx
for p in small do;
    >! p;
end;
```

---

## Maps

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

### Méthodes de Map

| Méthode          | Signature       | Retour    | Description                                     |
|------------------|-----------------|-----------|-------------------------------------------------|
| `.size()`        | `() → i`        | `i`       | Nombre de paires clé-valeur                     |
| `.get(key)`      | `(K) → V`       | `V`       | Retourne la valeur ; `halt.error` si clé absente|
| `.contains(key)` | `(K) → b`       | `b`       | Vérifie si la clé existe                        |
| `.insert(k, v)`  | `(K, V) → b`    | `b`       | Insère ou écrase                                |
| `.remove(key)`   | `(K) → b`       | `b`       | Supprime la paire ; `false` si clé absente      |
| `.keys()`        | `() → array:K`  | `array:K` | Retourne un tableau de clés                     |
| `.values()`      | `() → array:V`  | `array:V` | Retourne un tableau de valeurs                  |
| `.clear()`       | `() → b`        | `b`       | Supprime toutes les paires                      |
| `.toStr()`       | `() → s`        | `s`       | Sérialise la map en chaîne JSON                 |
| `.show()`        | `() → b`        | `b`       | Affiche le contenu dans le terminal             |
| `.toJson()`      | `() → json`     | `json`    | Sérialise la map en objet JSON                  |

Les clés de la map sont converties en chaînes dans l'objet JSON résultant.

### Sérialisation de Map (toJson)

#### Signature
`.toJson() → json`

#### Description
Sérialise la map en objet JSON. Toutes les clés sont converties en chaînes.

```xcx
map: scores {
    schema = [s <-> i]
    data = [ "alice" :: 100, "bob" :: 85 ]
};
json: j = scores.toJson();
```

**Sortie JSON :**
```json
{
    "alice": 100,
    "bob": 85
}
```

#### Correspondance de Types
| Type XCX | Type JSON |
|----------|-----------|
| `i`      | `number`  |
| `f`      | `number`  |
| `s`      | `string`  |
| `b`      | `boolean` |
| `date`   | `string` (format `"YYYY-MM-DD HH:mm:ss"`) |
| `json`   | (inchangé) |

Utilisez toujours `.contains()` avant `.get()` :

```xcx
if (ages.contains("alice")) then;
    >! ages.get("alice");
end;
```

---

## Tables

Structures de données relationnelles avec colonnes auto-incrémentées optionnelles.

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

Le modificateur `@auto` sur une colonne `i` crée un ID auto-incrémenté — il est ignoré dans `.insert()` et `.add()`.

> [!NOTE]
> Des attributs de colonne supplémentaires (`@pk`, `@unique`, `@optional`, `@default(v)`, `@fk(t.col)`) sont utilisés lors de la connexion d'une table à une base de données. Voir la [Documentation Base de Données](database.md) pour les détails.

### Accès aux Lignes

```xcx
products[0].name    --- "Laptop" (sugar for .get(0))
products[1].price   --- 1499.50
```

### Méthodes de Table

| Méthode              | Signature               | Retour  | Description                                          |
|----------------------|-------------------------|---------|------------------------------------------------------|
| `.count()`           | `() → i`                | `i`     | Nombre de lignes                                     |
| `.get(i)`            | `(i) → row`             | `row`   | Ligne à l'index `i`                                  |
| `.insert(vals...)`   | `(T...) → b`            | `b`     | Ajoute une ligne (ignore les colonnes `@auto`)       |
| `.add(vals...)`      | `(T...) → b`            | `b`     | Alias de `.insert()` — comportement identique        |
| `.update(i, vals)`   | `(i, [T...]) → b`       | `b`     | Remplace les valeurs de la ligne ; colonnes `@auto` préservées |
| `.delete(i)`         | `(i) → b`               | `b`     | Supprime la ligne à l'index `i`                      |
| `.where(pred)`       | `(expr) → table`        | `table` | Filtre — retourne une nouvelle table                 |
| `.join(t, pred)`     | `(table, pred) → table` | `table` | Jointure interne avec une autre table                |
| `.toJson()`          | `() → json`             | `json`  | Sérialise toutes les lignes en tableau JSON d'objets |
| `.show()`            | `() → b`                | `b`     | Affiche la table en format ASCII                     |

### Arguments Nommés pour `.add()` et `.insert()`

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

**Séparation des espaces de noms.** Le côté gauche de `=` est toujours le nom de colonne. Le côté droit est une expression de la portée locale. Ce sont deux espaces de noms indépendants — pas de conflit :

```xcx
s: name = "Alice";
users.add(name = name, age = 25);
--- left "name"  = column users.name
--- right "name" = local variable
```

### Filtrage (where)

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
> **Conflits de Noms dans `.where()` (S301)** : Les noms de colonnes ont la priorité sur les variables locales à l'intérieur des prédicats. Si une variable locale a le même nom qu'une colonne, renommez la variable pour éviter une erreur à la compilation.
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

### Jointures

```xcx
--- Key-based join
table: report = users.join(orders, "id", "user_id");

--- Lambda join
table: custom = tableA.join(tableB, (a, b) -> a.id == b.ref_id);
```

### Sérialisation (toJson)

#### Signature
`.toJson() → json`

#### Description
Sérialise toutes les lignes d'une table en tableau JSON, où chaque ligne devient un objet avec des clés correspondant aux noms de colonnes.

```xcx
table: products {
    columns = [ id :: i @auto, name :: s, price :: f ]
    rows = [ ("Laptop", 2999.99), ("Phone", 1499.50) ]
};

json: result = products.toJson();
```

**Sortie JSON :**
```json
[
    {"id": 1, "name": "Laptop", "price": 2999.99},
    {"id": 2, "name": "Phone",  "price": 1499.50}
]
```

#### Cas Particuliers
- **Table vide** : Retourne un tableau vide `[]`.
- **Table filtrée** : Seules les lignes actuellement dans la table (après `.where()`) sont sérialisées.
- **Table jointe** : Toutes les colonnes du résultat de la jointure sont incluses.