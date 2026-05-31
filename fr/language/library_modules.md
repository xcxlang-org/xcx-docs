# Bibliothèque Standard et Modules XCX 3.1

## Modules Intégrés

### crypto

Utilitaires de cryptographie.

| Méthode                              | Retour | Description                                      |
|--------------------------------------|--------|--------------------------------------------------|
| `crypto.hash(data, "bcrypt")`        | `s`    | Hache le mot de passe avec bcrypt                |
| `crypto.hash(data, "argon2")`        | `s`    | Hache le mot de passe avec argon2 (recommandé)   |
| `crypto.hash(data, "base64_encode")` | `s`    | Encode les données binaires/chaîne en Base64     |
| `crypto.hash(data, "base64_decode")` | `s`    | Décode une chaîne Base64 en binaire/chaîne       |
| `crypto.verify(password, hash, algo)`| `b`    | Retourne `true` si le mot de passe correspond    |
| `crypto.token(length)`               | `s`    | Génère un token hexadécimal aléatoire            |

```xcx
s: hash  = crypto.hash(password, "bcrypt");
s: hash2 = crypto.hash(password, "argon2");
b: valid = crypto.verify(password, hash2, "argon2");
s: token = crypto.token(32);
```

### store (E/S de Fichiers)

Tous les chemins doivent être **relatifs** à la racine du projet. Les chemins absolus ou les traversées de chemin (`..`) déclenchent `halt.fatal`.

| Méthode              | Signature                 | Retour   | Description                                       |
|----------------------|---------------------------|----------|---------------------------------------------------|
| `store.write(p, c)`  | `(s, s) → b`              | `b`      | Écrase le fichier. Crée les répertoires si besoin. |
| `store.read(p)`      | `(s) → s`                 | `s`      | Retourne le contenu ; `halt.fatal` si absent.      |
| `store.append(p, c)` | `(s, s) → b`              | `b`      | Ajoute au fichier. Crée si absent.                |
| `store.exists(p)`    | `(s) → b`                 | `b`      | Vérifie l'existence. Sans effets de bord.         |
| `store.delete(p)`    | `(s) → b`                 | `b`      | Supprime un fichier ou répertoire (récursif).     |
| `store.list(p)`      | `(s) → array:s`           | `array:s`| Retourne la liste des fichiers et dossiers.       |
| `store.isDir(p)`     | `(s) → b`                 | `b`      | `true` si le chemin est un répertoire.            |
| `store.size(p)`      | `(s) → i`                 | `i`      | Taille du fichier en octets.                      |
| `store.mkdir(p)`     | `(s) → b`                 | `b`      | Crée un répertoire (récursif).                    |
| `store.glob(pat)`    | `(s) → array:s`           | `array:s`| Retourne les fichiers correspondant au motif glob.|
| `store.zip(s, t)`    | `(s, s) → b`              | `b`      | Archive la source dans un zip cible.              |
| `store.unzip(z, d)`  | `(s, s) → b`              | `b`      | Extrait un zip vers la destination.               |

```xcx
store.write("log.txt", "First line");
store.append("log.txt", "\nSecond line");
s: content = store.read("log.txt");
if (store.exists("lock.pid")) then;
    >! "Already running";
end;
```

### env

| Méthode         | Signature      | Retour     | Description                                                  |
|-----------------|----------------|------------|--------------------------------------------------------------|
| `env.get(name)` | `(s) → s`      | `s`        | Retourne la valeur de la variable d'environnement ; `halt.error` si non définie |
| `env.args()`    | `() → array:s` | `array:s`  | Retourne les arguments CLI passés au programme               |

```xcx
s: db_url = env.get("DATABASE_URL");

array:s: args = env.args();
for arg in args do;
    >! arg;
end;
```

### random

| Méthode                            | Signature           | Retour | Description                                                                   |
|------------------------------------|---------------------|--------|-------------------------------------------------------------------------------|
| `random.choice from col`           | `(set/array) → T`   | `T`    | Sélectionne un élément aléatoire depuis l'ensemble ou le tableau fourni.      |
| `random.int(min, max @step num)`   | `(i, i, @i) → i`    | `i`    | Sélectionne un entier aléatoire dans `[min, max]`. Pas par défaut : `1`.      |
| `random.float(min, max @step num)` | `(f, f, @f) → f`    | `f`    | Sélectionne un flottant aléatoire dans `[min, max]`. Pas par défaut : `0.5`. |

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

## Système de Modules

### include

Fusionne le code d'un autre fichier dans l'espace de noms actuel.

```xcx
include "utils.xcx";
include "math.xcx" as m;

m.PI;
m.sqrt(16.0);
```

Sans alias — tous les symboles sont disponibles directement dans l'espace de noms actuel. Avec un alias — tous les symboles de premier niveau sont préfixés : `alias.symbol`.

Les dépendances cycliques sont détectées et rejetées à la compilation.