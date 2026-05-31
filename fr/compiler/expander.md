# Expandeur XCX — Documentation

> **Fichier :** `src/parser/expander.rs`  
> S'exécute **après** le parsage, **avant** l'analyse sémantique.

---

## Table des Matières

1. [Vue d'Ensemble](#vue-densemble)
2. [Responsabilités](#responsabilités)
3. [Résolution des include](#résolution-des-include)
4. [Préfixage des Alias](#préfixage-des-alias)
5. [Noms Protégés](#noms-protégés)
6. [Ordre de Recherche des Chemins d'Include](#ordre-de-recherche-des-chemins-dinclude)

---

## Vue d'Ensemble

L'Expandeur est une passe de réécriture d'AST séparée qui s'exécute après la phase de parsage et avant l'analyse sémantique. Il traite les directives `include` et `include ... as alias`.

```rust
pub struct Expander<'a> {
    interner:       &'a mut Interner,
    visiting_files: HashSet<PathBuf>,   // pour la détection de dépendances circulaires
    included_files: HashSet<PathBuf>,   // déduplication (chaque fichier une seule fois)
    aliases:        HashMap<StringId, String>,
    include_paths:  Vec<PathBuf>,       // chemins de recherche supplémentaires
}
```

---

## Responsabilités

### 1. Résolution des include
`include "file.xcx";` est remplacé par l'AST inliné de ce fichier.

- Les dépendances circulaires sont détectées via `visiting_files: HashSet<PathBuf>`
- Les fichiers sont dédupliqués via `included_files: HashSet<PathBuf>` (chaque fichier inclus une seule fois, sauf alias)

### 2. Préfixage des Alias
`include "math.xcx" as math;` fait que tous les noms de niveau supérieur de ce fichier reçoivent le préfixe `math.nom`. Les sites d'appel (`math.sin(x)`) sont réécrits de `MethodCall` vers `FunctionCall { name: "math.sin" }` via `expand_expr_inplace`.

Les fonctions `prefix_program()` / `prefix_stmt_impl()` / `prefix_expr_impl()` traversent l'ensemble du sous-AST, renommant toutes les références de symboles de niveau supérieur.

### 3. Préfixage des Noms de Fibres
Les références `FiberDecl::fiber_name` sont également préfixées, de sorte que les instanciations de fibres renommées soient correctement résolues après le préfixage.

### 4. Préfixage de YieldFrom
Les expressions `StmtKind::YieldFrom` sont parcourues de sorte que les appels de constructeurs de fibres à l'intérieur de `yield from` soient également renommés.

---

## Résolution des include

```xcx
include "utils.xcx";           --- Include simple (dédupliqué)
include "math.xcx" as math;    --- Include avec alias
```

Après un include avec alias :
- Tous les symboles de niveau supérieur de `math.xcx` reçoivent le préfixe `math.`
- Les appels à `math.sin(x)` sont réécrits de `MethodCall` vers `FunctionCall { name: "math.sin" }`

Exemple :

```xcx
--- math.xcx définit :
func sin(f: x -> f) { ... }
func cos(f: x -> f) { ... }

--- Après include "math.xcx" as math :
--- sin → math.sin
--- cos → math.cos
--- Appel math.sin(3.14) → FunctionCall("math.sin", [3.14])
```

---

## Préfixage des Alias

L'algorithme `prefix_stmt_impl` / `prefix_expr_impl` :

1. Collecte tous les noms de niveau supérieur du programme (`top_level_names: HashSet<StringId>`)
2. Pour chaque déclaration/expression : si l'identifiant appartient à `top_level_names`, il est remplacé par `prefix.nom`
3. Gère spécifiquement :
   - `FiberDecl::fiber_name` — pour que les instanciations fonctionnent après le renommage
   - `StmtKind::YieldFrom` — expressions de fibres dans `yield from`
   - `MethodCall` sur un objet alias → réécrit en `FunctionCall`

---

## Noms Protégés

Les noms suivants ne sont **jamais** préfixés (built-ins protégés) :

```
json    date    store   halt    terminal
net     env     crypto  EMPTY   math
random  i       f       s       b
from    main
```

---

## Ordre de Recherche des Chemins d'Include

1. Relatif au répertoire du fichier actuel
2. Dans le répertoire `lib/` (relatif au répertoire courant, puis en remontant depuis le chemin de l'exécutable)

Des chemins supplémentaires peuvent être ajoutés via :

```rust
expander.add_include_path(path);
```

Dans `main.rs`, le répertoire `lib/` relatif au répertoire courant est automatiquement ajouté :

```rust
if let Ok(cwd) = std::env::current_dir() {
    let lib_path = cwd.join("lib");
    if lib_path.exists() {
        expander.add_include_path(lib_path);
    }
}
```

---

## Détection des Erreurs

| Erreur | Description |
|---|---|
| `Circular dependency` | Boucle d'include détectée (`visiting_files`) |
| `File not found` | Le fichier `include` n'existe dans aucun des chemins de recherche |
| `Could not read file` | Erreur E/S lors de la lecture du fichier |