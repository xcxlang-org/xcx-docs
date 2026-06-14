# Manuel du gestionnaire de paquets PAX

PAX est le gestionnaire de paquets officiel de XCX, intégré directement dans le binaire `xcx`. Il gère les dépendances, la création de projets et l'automatisation des builds.

## Configuration du projet : `project.pax`

Tout projet PAX doit avoir un fichier `project.pax` à la racine de son répertoire. Il utilise un format déclaratif personnalisé.

```pax
---
PAX Project Configuration
*---
/
    name :: "my_project",
    deps :: [
        "user/repo",
        "https://example.com/lib.xcx"
    ]
/
```

- **name** : Nom logique du projet.
- **deps** : Liste des dépendances. Supporte les raccourcis GitHub (`user/repo`) et les URLs directes.

## Référence des commandes

PAX est invoqué via `xcx pax <commande>`.

| Commande                  | Description                                                        |
|---------------------------|--------------------------------------------------------------------|
| `xcx pax new <nom>`       | Génère une nouvelle structure de projet.                           |
| `xcx pax clone <paquet>`  | Télécharge un paquet publié en tant que projet local.              |
| `xcx pax install`         | Récupère les dépendances dans le répertoire `lib/`.                |
| `xcx pax add <dep>`       | Ajoute une dépendance et l'installe immédiatement.                 |
| `xcx pax remove <nom>`    | Supprime une dépendance de `project.pax`.                          |
| `xcx pax search <requête>`| Recherche les paquets disponibles dans le registre.                |
| `xcx pax run [chemin]`    | Exécute le projet (entrée : `src/main.xcx`).                       |

### `xcx pax clone <paquet>`

Télécharge un paquet publié depuis le registre PAX dans un nouveau répertoire local, prêt à être exécuté ou modifié.

```sh
xcx pax clone snake_game
xcx pax clone beta/snake_game
```

- L'argument est le nom du paquet tel qu'il figure dans le registre (optionnellement préfixé par `auteur/`).
- Crée un répertoire portant le nom du paquet dans le répertoire de travail courant.
- Copie la structure complète du projet : `project.pax`, `src/`, et tout autre fichier déclaré dans `files`.
- N'installe **pas** les dépendances automatiquement — exécuter `xcx pax install` dans le répertoire cloné ensuite.

```sh
xcx pax clone snake_game
cd snake_game
xcx pax install
xcx pax run
```

## Structure des répertoires

Un projet PAX standard suit cette organisation :
- `project.pax` : Configuration.
- `src/` : Code source (entrée principale : `main.xcx`).
- `lib/` : Dépendances téléchargées (gérées par PAX).
- `tests/` : Tests spécifiques au projet.