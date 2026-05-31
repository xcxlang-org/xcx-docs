# JSON et HTTP XCX 3.1

## JSON

Les objets JSON dans XCX 3.1 sont **mutables**.

### Création

```xcx
json: config <<< {"port": 8080, "debug": false} >>>;
json: user   <<< {"name": "", "age": 0} >>>;
```

Les valeurs dans le littéral peuvent être des espaces réservés (`""`, `0`, `false`) à remplir plus tard via `.set()`.

### Sérialisation depuis des Collections
Vous pouvez également créer des objets et tableaux JSON directement depuis des collections XCX en utilisant la méthode `.toJson()`. Disponible pour :
- **Maps** : Retourne un objet JSON.
- **Tables** : Retourne un tableau JSON d'objets.

Voir la [Documentation Collections](collections.md) pour plus de détails.

### Analyse (Parsing)

```xcx
json: parsed = json.parse(raw_string);
```

> [!CAUTION]
> **Panique sur JSON Invalide (R305)** : Si l'analyse échoue, la VM se termine immédiatement. Vérifiez le contenu de la chaîne avant d'analyser.

### Modèle de Mutabilité

Déclarez un schéma avec des valeurs zéro, puis remplissez :

```xcx
json: resp <<< {"token": "", "role": "", "uid": 0, "ok": false} >>>;
resp.set("token", crypto.token(32));
resp.set("role",  "admin");
resp.set("uid",   42);
resp.set("ok",    true);
yield net.respond(200, resp);
```

### Méthodes JSON

| Méthode                   | Signature             | Retour  | Description                                                        |
|---------------------------|-----------------------|---------|--------------------------------------------------------------------|
| `.exists(path)`           | `(s) → b`             | `b`     | Vérifie si le chemin existe et est non-null                        |
| `.get(path/idx)`          | `(s/i) → json`        | `json`  | Obtient l'élément au chemin ou index                               |
| `.bind(path, var)`        | `(s, ref) → b`        | `b`     | Extrait la valeur dans une variable XCX **pré-déclarée**           |
| `.set(path, val)`         | `(s, T) → b`          | `b`     | Définit la valeur au chemin ; crée la clé si absente               |
| `.push(val)`              | `(json) → b`          | `b`     | Ajoute un élément à un nœud tableau JSON                           |
| `.size()` / `.count()`    | `() → i`              | `i`     | Nombre de clés (objet) ou d'éléments (tableau)                     |
| `.toStr()`                | `() → s`              | `s`     | Sérialise en chaîne JSON                                           |
| `.inject(path, map, tbl)` | `(s, map, table) → b` | `b`     | Import en masse d'un tableau JSON dans une table XCX               |
| `.first()`                | `() → json`           | `json`  | Retourne le premier élément d'un tableau JSON ; halt.error si vide |

> [!NOTE]
> **`.push()` sur les tableaux JSON** : `.push()` fonctionne exclusivement sur les nœuds JSON qui sont des tableaux (`[]`). Appeler `.push()` sur un objet JSON (`{}`) déclenche `halt.error`.
>
> ```xcx
> json: data <<< {"items": []} >>>;
> json: obj <<< {"id": 1} >>>;
> data.get("items").push(obj);   --- OK: "items" is an array
> data.push(obj);                --- halt.error: data is an object, not an array
> ```

> [!IMPORTANT]
> **Syntaxe `.bind()`** : Le second argument doit être une **variable préalablement déclarée**. Vous ne pouvez pas déclarer le type inline.
>
> ```xcx
> --- Wrong
> req.bind("ip", s: ip);
>
> --- Correct
> s: ip;
> req.bind("ip", ip);
> ```

### Notation de Chemin

La notation pointée et la notation entre crochets sont toutes deux supportées pour l'accès imbriqué :

```xcx
--- Nested field
json: cfg <<< {"server": {"host": "localhost"}} >>>;
s: host;
cfg.bind("server.host", host);

--- Array index
json: resp <<< {"items": []} >>>;
resp.set("items[0]", first_item);
resp.set("items[1]", second_item);
```

### .inject() — Import en Masse

Importez un tableau JSON directement dans une table XCX :

```xcx
json: data <<< {"users":[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]} >>>;
table: imported { columns=[uid::i, uname::s] rows=[EMPTY] };
map: mapping { schema=[s<->s] data=["uid"::"id", "uname"::"name"] };
data.inject("users", mapping, imported);
imported.show();
```

### JSON dans les Requêtes HTTP

À l'intérieur des gestionnaires HTTP, `json: req` a cette structure :

```json
{
    "method":  "POST",
    "path":    "/api/login",
    "query":   { "page": "1" },
    "headers": { "authorization": "Bearer ..." },
    "body":    { ... },
    "ip":      "1.2.3.4"
}
```

```xcx
s: ip;
req.bind("ip", ip);

json: body;
req.bind("body", body);

json: headers;
req.bind("headers", headers);

s: auth;
headers.bind("authorization", auth);
```

---

## Client HTTP

### API Haut Niveau

| Méthode                | Signature          | Retour  |
|------------------------|--------------------|---------|
| `net.get(url)`         | `(s) → json`       | `json`  |
| `net.post(url, body)`  | `(s, json) → json` | `json`  |
| `net.put(url, body)`   | `(s, json) → json` | `json`  |
| `net.delete(url)`      | `(s) → json`       | `json`  |

```xcx
json: resp = net.get("https://api.example.com/users");

json: body <<< {"name": "Alice"} >>>;
json: resp = net.post("https://api.example.com/users", body);
```

### Objet de Réponse

```json
{
    "status":  200,
    "ok":      true,
    "body":    { ... },
    "headers": { "content-type": "application/json" },
    "text":    "...",
    "error":   "..."
}
```

| Champ     | Type   | Description                                                        |
|-----------|--------|--------------------------------------------------------------------|
| `status`  | `i`    | Code de statut HTTP                                                |
| `ok`      | `b`    | `true` quand le statut est >= 200 et < 300                         |
| `body`    | `json` | Corps de réponse analysé (si JSON) ; sinon `null`                  |
| `headers` | `json` | En-têtes de réponse                                                |
| `text`    | `s`    | Corps de réponse en chaîne brute (toujours disponible)             |
| `error`   | `s`    | Message d'erreur si la requête a échoué ; chaîne vide si OK        |

Vérifiez toujours `ok` avant d'accéder à `body` :

```xcx
json: resp = net.get("https://api.example.com/data");
if (resp.ok) then;
    --- working with resp.body
else;
    s: err;
    if (resp.exists("error")) then;
        resp.bind("error", err);
    end;
    >! "Error: " + s(resp.get("status")) + " | " + err;
end;
```

### Constructeur Bas Niveau (`net.request`)

```xcx
net.request {
    method  = "POST",
    url     = "https://api.example.com/data",
    headers = ["Authorization" :: "Bearer xyz", "X-App" :: "XCX"],
    body    = my_json,
    timeout = 5000
} as resp;
```

| Champ     | Type        | Requis | Défaut   | Description                            |
|-----------|-------------|--------|----------|----------------------------------------|
| `method`  | `s`         | Oui    | —        | `"GET"`, `"POST"`, `"PUT"`, etc.       |
| `url`     | `s`         | Oui    | —        | URL complète avec schéma               |
| `headers` | `map:s<->s` | Non    | `{}`     | En-têtes de requête supplémentaires    |
| `body`    | `json`      | Non    | `null`   | Ignoré pour GET et DELETE              |
| `timeout` | `i`         | Non    | `10000`  | Millisecondes                          |

---

## Serveur HTTP

### Directive serve

```xcx
serve: app {
    port    = 8080,
    host    = "0.0.0.0",
    workers = 4,
    routes  = [
        "POST   /api/login"     :: handle_login,
        "GET    /api/user"      :: handle_user,
        "DELETE /api/users"     :: handle_delete,
        "OPTIONS *"             :: handle_options,
        "*"                     :: handle_404
    ]
};
```

| Champ     | Type         | Requis | Défaut        | Description                                    |
|-----------|--------------|--------|---------------|------------------------------------------------|
| `port`    | `i`          | Oui    | —             | Numéro de port (1–65535)                       |
| `host`    | `s`          | Non    | `"127.0.0.1"` | `"0.0.0.0"` = toutes les interfaces            |
| `workers` | `i`          | Non    | `1`           | Fibres par requête                             |
| `routes`  | liste routes | Oui    | —             | Vérifié de haut en bas, premier match gagne    |

> [!NOTE]
> `serve:` est une **instruction terminale**. Aucun code après cette directive ne s'exécutera.
> Le joker `*` correspond à toute méthode ou chemin — placez-le toujours en dernier.

### Fibres Gestionnaires

Chaque gestionnaire doit être une fibre avec la signature `fiber name(json: req -> json)` :

```xcx
fiber handle_health(json: req -> json) {
    yield net.respond(200, <<< {"status": "ok"} >>>);
};
```

Un gestionnaire qui n'appelle pas `yield net.respond(...)` déclenche automatiquement une erreur `500 Internal Server Error`.

### net.respond()

```xcx
yield net.respond(200, my_json);
yield net.respond(201, my_json, ["Location" :: "/users/42"]);
yield net.respond(204, <<< {} >>>);
yield net.respond(404, <<< {"error": "not found"} >>>);
```

| Paramètre | Type        | Requis | Description                              |
|-----------|-------------|--------|------------------------------------------|
| status    | `i`         | Oui    | Code de statut HTTP                      |
| body      | `json \| s` | Oui    | Objet JSON ou chaîne brute               |
| headers   | `map:s<->s` | Non    | En-têtes de réponse supplémentaires      |

### CORS et Preflight

Par défaut, le moteur XCX fournit un support CORS automatique en ajoutant ces en-têtes s'ils sont absents de la réponse :
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

> [!TIP]
> **Bonne Pratique** : Bien que le moteur fournisse des valeurs par défaut, il est **fortement recommandé** de déclarer explicitement ces en-têtes dans votre code.

Pour le support preflight (`OPTIONS`), utilisez un gestionnaire dédié :

```xcx
fiber handle_options(json: req -> json) {
    yield net.respond(204, <<< {} >>>, [
        {
            "Access-Control-Allow-Methods": "GET, POST, DELETE, PATCH, OPTIONS",
            "Access-Control-Allow-Headers": "Content-Type, Authorization, X-CSRF-TOKEN"
        }
    ]);
};
```

### Requêtes Concurrentes

Les fibres permettent des E/S en parallèle :

```xcx
fiber fetch(s: url -> json) {
    yield net.get(url);
};

fiber:json: f1 = fetch("https://api.example.com/users");
fiber:json: f2 = fetch("https://api.example.com/posts");
json: r1 = f1.next();
json: r2 = f2.next();
```

### Contraintes de Sécurité

| Contrainte                      | Comportement                                        |
|---------------------------------|-----------------------------------------------------|
| `localhost` / `127.0.0.1`       | Autorisé par défaut                                 |
| `169.254.x.x` (lien local)      | `halt.fatal` — protection SSRF                      |
| `10.x`, `172.16.x`, `192.168.x` | Bloqué en mode production                           |
| URLs `file://`                  | `halt.fatal`                                        |
| Taille max du corps de réponse  | 10 Mo                                               |
| Corps de requête entrante max   | 10 Mo — retourne 413 sans invoquer le gestionnaire  |