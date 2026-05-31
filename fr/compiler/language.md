# Référence du Langage XCX — v3.1

> Référence complète de la syntaxe et de la sémantique du langage XCX.

---

## Table des Matières

1. [Types de Données](#types-de-données)
2. [Variables et Constantes](#variables-et-constantes)
3. [Opérateurs](#opérateurs)
4. [Flux de Contrôle](#flux-de-contrôle)
5. [Fonctions](#fonctions)
6. [Fibres (Coroutines)](#fibres-coroutines)
7. [Collections](#collections)
8. [JSON](#json)
9. [Tables](#tables)
10. [Base de Données (SQLite)](#base-de-données-sqlite)
11. [Réseau (HTTP)](#réseau-http)
12. [Stockage (Fichiers)](#stockage-fichiers)
13. [Terminal](#terminal)
14. [Date et Heure](#date-et-heure)
15. [Aléatoire](#aléatoire)
16. [Cryptographie](#cryptographie)
17. [Diagnostics et Arrêt](#diagnostics-et-arrêt)
18. [Include](#include)

---

## Types de Données

| Type XCX | Description | Exemple |
|---|---|---|
| `i` | Entier 48 bits | `i: age = 25;` |
| `f` | Flottant 64 bits | `f: pi = 3.14;` |
| `s` | Chaîne UTF-8 | `s: name = "Alice";` |
| `b` | Booléen | `b: flag = true;` |
| `date` | Date/Heure (ms) | `date: d = date("2024-01-01");` |
| `json` | JSON quelconque | `json: data = <<<{}>>>;` |
| `array:T` | Tableau typé | `array:i nums = [1, 2, 3];` |
| `set:N/Q/Z/S/C/B` | Ensemble | `set:N mySet = set:N { 1,,100 };` |
| `map:K<->V` | Map | `map:s<->i scores = ["Alice" :: 100];` |
| `table` | Table Relationnelle | `table: ...` |
| `fiber:T` | Coroutine | `fiber:i f = myFiber(10);` |
| `database` | Connexion SQLite | `database: db = database { ... };` |

### Conversion de Types

```xcx
i(x)   --- vers entier
f(x)   --- vers flottant
s(x)   --- vers chaîne
b(x)   --- vers booléen
```

---

## Variables et Constantes

```xcx
--- Déclaration avec type explicite
i: count = 0;
s: greeting = "Hello";

--- Constante (ne peut pas être réassignée)
const f: PI = 3.14159;

--- Inférence de Type
var result = 42;        --- type inféré comme int
var name = "Bob";       --- type inféré comme string
```

### Assignation

```xcx
count = count + 1;
greeting = "World";
```

---

## Opérateurs

### Arithmétique

| Opérateur | Description | Exemple |
|---|---|---|
| `+` | Addition / Concaténation de chaînes | `a + b` |
| `-` | Soustraction | `a - b` |
| `*` | Multiplication | `a * b` |
| `/` | Division | `a / b` |
| `%` | Modulo | `a % b` |
| `^` | Exponentiation | `a ^ b` |
| `++` | Concaténation de chiffres | `12 ++ 34` = `1234` |

### Comparaisons

`==`, `!=`, `>`, `<`, `>=`, `<=`

### Logique

| Opérateur | Alternative | Description |
|---|---|---|
| `AND` | `&&` | Et logique |
| `OR` | `||` | Ou logique |
| `NOT` | `!!` | Non logique |
| `HAS` | — | Appartenance (`"abc" HAS "b"`) |

### Opérateurs d'Ensembles

| Opérateur | Symbole | Unicode |
|---|---|---|
| `UNION` | | `∪` |
| `INTERSECTION` | | `∩` |
| `DIFFERENCE` | `\` | |
| `SYMMETRIC_DIFFERENCE` | | `⊕` |

### Opérateurs de Date

```xcx
date("2024-01-15") + 7    --- date + jours
date("2024-01-15") - 7    --- date - jours
date("2024-01-15") - date("2024-01-01")  --- différence en ms
```

---

## Flux de Contrôle

### If / ElseIf / Else

```xcx
if (condition) then;
    ...
elseif (autre_condition) then;
    ...
else;
    ...
end;
```

### While

```xcx
while (condition) do;
    ...
end;
```

### For (Plage Numérique)

```xcx
for i in 1 to 100 do;
    >! i;
end;

--- Avec pas
for i in 0 to 100 @step 5 do;
    >! i;
end;
```

### For (Tableau / Ensemble / Fibre)

```xcx
for item in myArray do;
    >! item;
end;

for elem in mySet do;
    >! elem;
end;

for val in myFiber do;
    >! val;
end;
```

### Break et Continue

```xcx
while (true) do;
    if (condition) then;
        break;
    end;
    if (skip) then;
        continue;
    end;
end;
```

---

## Fonctions

### Style Accolades (C-like)

```xcx
func add(i: a, i: b -> i) {
    return a + b;
}
```

### Style XCX (Bloc de mots-clés)

```xcx
func:i: add(i: a, i: b) do;
    return a + b;
end;
```

### Lambdas

```xcx
var double = x -> x * 2;
var sum = (x, y) -> x + y;
```

### Appel

```xcx
i: result = add(3, 4);
```

---

## Fibres (Coroutines)

Les fibres sont des coroutines coopératives — pas des threads OS.

### Définition de Fibre

```xcx
fiber counter(i: start) {
    i: n = start;
    while (true) do;
        yield n;
        n = n + 1;
    end;
}
```

### Instanciation de Fibre

```xcx
fiber:i: c = counter(1);
```

### Itération sur une Fibre

```xcx
for val in c do;
    >! val;
    if (val >= 10) then;
        break;
    end;
end;
```

### Reprise Manuelle

```xcx
i: val = c.next();
b: done = c.isDone();
c.close();
```

### Délégation de Yield

```xcx
fiber pipeline(array:i data) {
    yield from processData(data);
}
```

### Fibre Void (sans valeur)

```xcx
fiber doWork() {
    --- faire du travail
    yield;  --- suspension sans valeur
}

fiber:void: w = doWork();
w.run();
```

---

## Collections

### Tableau

```xcx
array:i nums = [1, 2, 3, 4, 5];
nums.push(6);
i: first = nums.get(0);
nums.set(0, 99);
nums.delete(0);
i: size = nums.size();
b: has = nums.contains(3);
nums.sort();
nums.reverse();
s: joined = nums.join(", ");
```

### Ensemble

```xcx
--- Ensemble naturel avec plage
set:N primes = set:N { 2, 3, 5, 7, 11 };

--- Ensemble avec plage
set:Z mySet = set:Z { -10,,10 };

--- Ensemble avec pas
set:N evens = set:N { 2,,100 @step 2 };

--- Opérations
primes.add(13);
primes.remove(2);
b: hasFive = primes.contains(5);
i: count = primes.size();

--- Opérations d'ensembles
set:N union = a UNION b;
set:N inter = a INTERSECTION b;
set:N diff  = a DIFFERENCE b;
set:N sym   = a SYMMETRIC_DIFFERENCE b;
```

### Map

```xcx
map:s<->i scores = ["Alice" :: 100, "Bob" :: 85];
scores.set("Charlie", 90);
i: aliceScore = scores.get("Alice");
scores.remove("Bob");
array:s keys = scores.keys();
```

---

## JSON

```xcx
--- Parsage JSON
json: data = json.parse(jsonString);

--- Bloc brut (JSON inline)
json: config = <<<{ "host": "localhost", "port": 8080 }>>>;

--- Accès aux Champs
json: host = data.host;
json: nested = data.get("/server/host");

--- Modification
data.set("/port", 9090);
data.push("/items", "newItem");

--- Liaison à des variables
s: hostname;
data.bind("/host", hostname);

--- Injection dans une table
data.inject(mapping, myTable);

--- Vérification
b: exists = data.exists("/optional");
i: count = data.size();
```

---

## Tables

```xcx
--- Définition de Table
table: users = table {
    columns = [
        id   :: i @auto,
        name :: s,
        age  :: i,
        email :: s
    ]
    rows = EMPTY
};

--- Insertion
users.insert("Alice", 30, "alice@example.com");
users.insert(name = "Bob", age = 25, email = "bob@example.com");

--- Requête
table: adults = users.where(row -> row.age >= 18);
table: result = users.join(orders, "id", "userId");

--- Affichage
users.show();

--- Lignes
i: count = users.count();
json: row = users.get(0);

--- Mise à jour
users.update(0, ["Alice Updated", 31, "alice@new.com"]);

--- Suppression
users.delete(0);

--- Conversion en JSON
json: usersJson = users.toJson();

--- Sauvegarde (@pk requis pour save())
table: products = table {
    columns = [id :: i @pk, name :: s, price :: f]
    rows = EMPTY
};
products.save("Laptop", 999.99);  --- insert ou update
```

---

## Base de Données (SQLite)

```xcx
--- Connexion
database: db = database {
    engine = "sqlite",
    path = "myapp.db",
    users = users,
    products = products
};

--- Le schéma est automatiquement synchronisé

--- Insertion
db.insert(users, "Alice", 30, "alice@example.com");

--- Requête
table: result = db.fetch(users);

--- Requête filtrée
table: adults = db.fetch(users.where(row -> row.age >= 18));

--- SQL brut
json: rows = db.queryRaw("SELECT * FROM users WHERE age > 25");

--- SQL paramétré
json: res = db.exec("INSERT INTO logs (msg) VALUES (?)", ["event"]);

--- Transactions
db.begin();
db.insert(users, "Charlie", 20, "charlie@example.com");
db.commit();

--- Suppression (requiert .where())
db.remove(users).where(row -> row.age < 18);

--- Schéma
db.sync(users);
db.drop(users);
db.truncate(users);
```

---

## Réseau (HTTP)

### Requêtes HTTP Simples

```xcx
--- GET
json: resp = net.get("https://api.example.com/data");

--- POST
json: resp = net.post("https://api.example.com/users", payload);

--- PUT, DELETE, PATCH
json: resp = net.put("https://api.example.com/users/1", data);
json: resp = net.delete("https://api.example.com/users/1");
```

### Requête Complète

```xcx
net.request {
    method = "POST",
    url = "https://api.example.com/users",
    headers = ["Authorization" :: "Bearer token123"],
    body = payload,
    timeout = 5000
} as resp;

--- Vérification de la réponse
b: ok = resp.ok;
i: status = resp.status;
json: body = resp.body;
```

### Serveur HTTP

```xcx
fiber getHandler(json: req) {
    s: path = req.url;
    json: response = <<<{ "message": "Hello World" }>>>;
    net.respond(200, response);
}

serve: myServer {
    port = 8080,
    host = "0.0.0.0",
    workers = 4,
    routes = [
        ["GET /"    :: getHandler],
        ["POST /api" :: postHandler]
    ]
};
```

---

## Stockage (Fichiers)

```xcx
--- Lecture / Écriture
s: content = store.read("data.txt");
store.write("output.txt", "Hello World");
store.append("log.txt", "new line\n");

--- Vérification
b: exists = store.exists("file.txt");
i: size = store.size("file.txt");
b: isDir = store.isDir("path/");

--- Opérations sur les Répertoires
store.mkdir("newdir");
array:s files = store.list("mydir");
array:s matches = store.glob("*.xcx");

--- Suppression
store.delete("temp.txt");

--- Archivage
b: zipped = store.zip("folder", "archive.zip");
b: ok = store.unzip("archive.zip", "output/");
```

---

## Terminal

```xcx
--- Impression sans saut de ligne
.terminal!write "Hello ";
.terminal!write "World\n";

--- Effacer l'écran
.terminal!clear;

--- Quitter le programme
.terminal!exit;

--- Exécuter une commande système
b: ok = .terminal!run "ls -la";

--- Mode raw (pour les applications interactives)
.terminal!raw;

--- Lire une touche (non-bloquant)
s: key = input.key();

--- Lire une touche (bloquant)
s: key = input.key() @wait;

--- Vérifier la disponibilité de l'entrée
b: ready = input.ready();

--- Curseur
.terminal!cursor on;
.terminal!cursor off;
.terminal!move 10 5;    --- colonne 10, ligne 5

--- Retourner au mode normal
.terminal!normal;
```

---

## Date et Heure

```xcx
--- Date actuelle
date: now = date.now();

--- Littéral de Date
date: d = date("2024-01-15");
date: d2 = date("15/01/2024", "DD/MM/YYYY");

--- Composants
i: year = d.year();
i: month = d.month();
i: day = d.day();
i: hour = d.hour();
i: minute = d.minute();
i: second = d.second();

--- Formatage
s: formatted = d.format("YYYY-MM-DD");
s: withTime = d.format("YYYY-MM-DD HH:mm:ss");

--- Arithmétique
date: tomorrow = d + 1;
date: yesterday = d - 1;
i: diff = date("2024-12-31") - date("2024-01-01");
```

---

## Aléatoire

```xcx
--- Entier aléatoire (plage)
i: n = random.int(1, 100);

--- Entier aléatoire avec pas
i: even = random.int(2, 100 @step 2);

--- Flottant aléatoire
f: x = random.float(0.0, 1.0);

--- Flottant aléatoire avec pas
f: y = random.float(0.0, 10.0 @step 0.5);

--- Élément aléatoire depuis une collection
array:s colors = ["red", "green", "blue"];
s: picked = random.choice from colors;

set:N nums = set:N { 1,,10 };
i: val = random.choice from nums;
```

---

## Cryptographie

```xcx
--- Hachage
s: hash = crypto.hash(password, "bcrypt");
s: hash2 = crypto.hash(password, "argon2");

--- Vérification
b: ok = crypto.verify(password, hash, "bcrypt");
b: ok2 = crypto.verify(password, hash2, "argon2");

--- Token aléatoire (hex)
s: token = crypto.token(32);   --- 32 caractères hexadécimaux

--- Base64
s: encoded = crypto.hash(data, "base64_encode");
s: decoded = crypto.hash(data, "base64_decode");
```

---

## Diagnostics et Arrêt

```xcx
--- Impression (>!)
>! "Hello World";
>! age;
>! "Valeur : " + s(count);

--- Entrée (>?)
>? name;       --- lit dans une variable existante
>? age;

--- Niveaux d'arrêt
halt.alert >! "Avertissement";          --- continue l'exécution
halt.error >! "Erreur Logique";         --- arrête le frame
halt.fatal >! "Erreur Critique";        --- arrête le frame

--- Attente
@wait(1000);     --- attendre 1 seconde (ms)
@wait 500;       --- syntaxe alternative
```

---

## Include

```xcx
--- Include simple (dédupliqué — une seule fois)
include "utils.xcx";

--- Include avec alias (tous les noms reçoivent un préfixe)
include "math.xcx" as math;

--- Utilisation après include avec alias
f: result = math.sin(3.14);
f: cosVal = math.cos(0.0);
```

### Chemins de Recherche

1. Relatif au répertoire du fichier actuel
2. Dans le répertoire `lib/` (répertoire courant et chemins de bibliothèque XCX)

---

## Variables d'Environnement et CLI

```xcx
--- Variable d'Environnement
s: apiKey = env.get("API_KEY");

--- Arguments de Ligne de Commande
array:s args = env.args();
s: firstArg = args.get(0);
```

---

## Commentaires

```xcx
--- Commentaire sur une ligne

---
Commentaire
multi-lignes
*---
```