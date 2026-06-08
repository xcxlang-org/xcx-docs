# Manuel du Gestionnaire de Paquets PAX

PAX est le gestionnaire de paquets officiel de XCX, intégré directement dans le binaire `xcx`. Il gère les dépendances, l'échafaudage de projets, la publication dans le registre et l'automatisation des builds.

---

## Configuration du projet : `project.pax`

Chaque projet PAX doit contenir un fichier `project.pax` dans son répertoire racine. Il utilise un format déclaratif personnalisé.

```pax
---
PAX Project Configuration
*---
/
    name        :: "my_project",
    version     :: "1.0.0",
    author      :: "yourname",
    description :: "A short description of the project",
    main        :: "src/main.xcx",
    tags  :: ["tag1", "tag2"],
    files :: ["src/main.xcx", "src/lib.xcx"],
    deps  :: [
        "somelib@1.0.0",
        "author/repo@latest",
        "https://example.com/lib.xcx"
    ]
/
```

### Champs

| Champ         | Requis | Description                                                                  |
|---------------|--------|------------------------------------------------------------------------------|
| `name`        | Oui    | Nom logique du projet/paquet.                                                |
| `version`     | Oui    | Chaîne de version sémantique (ex. `"1.0.0"`).                               |
| `author`      | Non    | Nom de l'auteur, affiché dans le registre.                                   |
| `description` | Non    | Courte description du projet.                                                |
| `main`        | Non    | Point d'entrée personnalisé. Par défaut `src/main.xcx` si omis.             |
| `tags`        | Non    | Liste de tags pour la découvrabilité dans le registre.                       |
| `files`       | Non    | Liste explicite des fichiers à inclure lors de la publication. Si omis, PAX exclut automatiquement les fichiers non-code. |
| `deps`        | Non    | Liste des dépendances. Supporte les noms versionnés, les raccourcis `author/repo` et les URLs directes. |

### Formats de dépendances

```pax
deps :: [
    "mylib@1.0.0",           --- paquet du registre avec version spécifique
    "mylib@latest",          --- dernière version du registre
    "author/mylib@2.1.0",    --- avec préfixe d'auteur
    "https://example.com/lib.xcx"  --- URL directe
]
```

---

## Fichier de verrouillage : `pax.lock`

PAX maintient automatiquement un fichier `pax.lock` après chaque installation. Il enregistre la version résolue exacte et le chemin local de chaque dépendance installée.

```json
{
  "mylib": { "version": "mylib@1.0.0", "file": "lib/mylib/" },
  "otherlib": { "version": "otherlib@latest", "file": "lib/otherlib/" }
}
```

- Ne **pas** modifier `pax.lock` manuellement.
- Committez-le dans le contrôle de version pour des builds reproductibles.

---

## Structure des répertoires

Un projet PAX standard suit cette organisation :

```
my_project/
├── project.pax       # Configuration du projet
├── pax.lock          # Fichier de verrouillage autogénéré
├── src/
│   └── main.xcx      # Point d'entrée principal
├── lib/              # Dépendances installées (gérées par PAX)
└── tests/            # Tests spécifiques au projet
```

---

## Référence des commandes

PAX s'invoque via `xcx pax <commande>`.

| Commande                     | Description                                            |
|------------------------------|--------------------------------------------------------|
| `xcx pax new <name>`         | Créer une nouvelle structure de projet.                |
| `xcx pax install`            | Télécharger toutes les dépendances dans `lib/`.        |
| `xcx pax add <dep>`          | Ajouter une dépendance et l'installer immédiatement.   |
| `xcx pax remove <name>`      | Supprimer une dépendance de `project.pax`.             |
| `xcx pax clone <package>`    | Télécharger un paquet publié comme projet local.       |
| `xcx pax run [path]`         | Exécuter le point d'entrée du projet.                  |
| `xcx pax search <query>`     | Rechercher des paquets disponibles dans le registre.   |
| `xcx pax login <token>`      | Enregistrer un token d'authentification du registre.   |
| `xcx pax logout`             | Supprimer le token d'authentification enregistré.      |
| `xcx pax whoami`             | Vérifier le compte actuel du registre.                 |
| `xcx pax publish`            | Publier le projet dans le registre PAX.                |

---

## Détails des commandes

### `xcx pax new <name>`

Crée un nouveau projet avec la structure de répertoires standard.

```sh
xcx pax new my_project
```

Crée :
```
my_project/
├── project.pax
├── src/
│   └── main.xcx
└── README.md
```

---

### `xcx pax install`

Lit `project.pax`, résout toutes les `deps` et les télécharge dans `lib/`. Ignore les paquets déjà présents (par nom).

```sh
xcx pax install
```

- Installe uniquement les fichiers déclarés dans le champ `files` de la dépendance.
- Si la dépendance n'a pas de champ `files`, PAX exclut automatiquement `README.md`, `LICENSE`, `tests/`, `docs/`, `.git*` et `project.pax`.
- Met à jour `pax.lock` après chaque téléchargement réussi.

---

### `xcx pax add <dep>`

Ajoute une dépendance à `project.pax` et lance immédiatement l'installation.

```sh
xcx pax add mylib@1.0.0
xcx pax add author/mylib@latest
```

---

### `xcx pax remove <name>`

Supprime une entrée de dépendance de `project.pax`.

```sh
xcx pax remove mylib
```

> **Note :** Les dossiers `lib/` et les entrées `pax.lock` ne sont pas nettoyés automatiquement. Relancez `xcx pax install` ou supprimez-les manuellement.

---

### `xcx pax clone <package>`

Télécharge un paquet publié du registre PAX comme projet local complet, prêt à être exécuté ou modifié.

```sh
xcx pax clone snake_game
xcx pax clone author/snake_game
```

- Crée un répertoire nommé d'après le paquet dans le répertoire de travail courant.
- Copie la structure complète du projet, y compris `project.pax`, `src/` et tous les fichiers déclarés.
- **N'installe pas** les dépendances automatiquement.

**Flux de travail typique :**
```sh
xcx pax clone snake_game
cd snake_game
xcx pax install
xcx pax run
```

---

### `xcx pax run [path]`

Exécute le projet. Ordre de résolution du point d'entrée :

1. Chemin fourni comme argument (ex. `xcx pax run other/main.xcx`)
2. Champ `main` dans `project.pax`
3. Par défaut : `src/main.xcx`

```sh
xcx pax run
xcx pax run src/alt.xcx
```

---

### `xcx pax search <query>`

Recherche dans le registre PAX les paquets correspondant à la requête.

```sh
xcx pax search json
```

Format de sortie :
```
- json_utils (v1.2.0) by alice
- fast_json (v0.9.1) by bob
```

---

### `xcx pax login <token>`

Enregistre un token d'authentification du registre localement dans `.pax_token`.

```sh
xcx pax login your_token_here
```

---

### `xcx pax logout`

Supprime le token d'authentification enregistré.

```sh
xcx pax logout
```

---

### `xcx pax whoami`

Vérifie la session actuelle du registre et affiche les informations du compte.

```sh
xcx pax whoami
# Account: alice [developer]
```

---

### `xcx pax publish`

Publie le projet actuel dans le registre PAX. Nécessite une connexion préalable.

```sh
xcx pax publish
```

- Lit les métadonnées de `project.pax` (`name`, `version`, `description`).
- Scanne et envoie tous les fichiers du projet (excluant `.zip`, `.pax_token`, `.git`).
- Envoie une archive compressée et un manifeste de fichiers au registre.

> **Note :** Incrémentez `version` dans `project.pax` avant chaque publication. Le registre peut rejeter les versions en double.

---

## Configuration du registre

Par défaut PAX se connecte à `pax.xcxlang.com`. Vous pouvez le remplacer via un fichier `.pax_config` à la racine du projet :

```json
{
  "registry": "https://my-custom-registry.example.com"
}
```

- HTTP est utilisé automatiquement pour `localhost` et `127.0.0.1`.
- HTTPS est utilisé pour tous les autres hôtes.
