# Analyse Sémantique XCX (Sema) — Documentation

> **Fichiers :** `src/sema/checker.rs`, `src/sema/symbol_table.rs`, `src/sema/interner.rs`

---

## Table des Matières

1. [Vue d'Ensemble](#vue-densemble)
2. [Internement des Chaînes](#internement-des-chaînes)
3. [Table des Symboles](#table-des-symboles)
4. [Vérificateur de Types](#vérificateur-de-types)
5. [Règles de Compatibilité des Types](#règles-de-compatibilité-des-types)
6. [Codes d'Erreur](#codes-derreur)
7. [Rapport d'Erreurs](#rapport-derreurs)

---

## Vue d'Ensemble

La phase Sema valide l'AST pour la cohérence logique et de type avant la génération du bytecode. Elle se compose de trois composants :

```
Interner → StringId (u32)
     ↓
SymbolTable → portées de types hiérarchiques
     ↓
Checker → collection de TypeErrors
```

Le programme n'est compilé que si le `Vec<TypeError>` résultant est vide.

---

## Internement des Chaînes

**Fichier :** `src/sema/interner.rs`

L'`Interner` mappe `&str → StringId(u32)`. C'est la source unique de vérité pour toutes les identités de chaînes dans le compilateur. Il est créé pendant le lexage/parsage et passé par référence au vérificateur et au compilateur.

```rust
Interner::intern("foo") → StringId(42)
Interner::lookup(StringId(42)) → "foo"
```

**Implémentation :**

```rust
pub struct Interner {
    map:     HashMap<String, StringId>,
    strings: Vec<String>,
}
```

Chaque chaîne unique est stockée une fois dans `strings: Vec<String>`. Le reste du pipeline travaille avec des IDs numériques, éliminant les comparaisons sur le tas pendant la vérification des types et la compilation.

---

## Table des Symboles

**Fichier :** `src/sema/symbol_table.rs`

La `SymbolTable` gère les liaisons de variables dans des portées imbriquées en utilisant une **chaîne de pointeurs parents**.

### Structure

```rust
pub struct SymbolTable<'a> {
    parent: Option<&'a SymbolTable<'a>>,  // référence à la portée englobante
    scopes: Vec<HashMap<String, Type>>,   // pile de frames de portée (local)
    consts: Vec<HashSet<String>>,         // quels noms sont const, par frame
}
```

### Création d'une Portée Enfant

Lors de l'entrée dans un corps de fonction ou de fibre, une nouvelle `SymbolTable` est créée avec une référence à la table englobante au lieu de la cloner :

```rust
let mut func_symbols = SymbolTable::new_with_parent(symbols);
```

C'est O(1) — seulement une `HashMap` vide est allouée pour le nouveau frame. Les recherches traversent la chaîne parente au besoin.

### Cycle de Vie des Portées

| Méthode | Description |
|---|---|
| `enter_scope()` | Pousse un nouveau frame — pour les corps `if`, `while`, `for` |
| `exit_scope()` | Sort de la portée actuelle |
| `new_with_parent(parent)` | Crée une table enfant — pour les corps de fonction/fibre |
| `define(name, ty, is_const)` | Toujours écrit dans la portée **la plus interne** |
| `lookup(name)` | Traverse de l'intérieur vers l'extérieur, puis vers le parent |
| `has_in_current_scope(name)` | Vérifie seulement le frame le plus interne — pour la détection de redéclaration |
| `is_const(name)` | Vérifie si la portée propriétaire a marqué le nom comme const |

### Pas de Masquage de Variables

XCX **ne supporte pas** le masquage de variables. Définir une variable qui existe déjà dans la **portée actuelle** retourne `RedefinedVariable`. Les variables dans les portées parentes sont accessibles mais ne peuvent pas être redéclarées dans une portée enfant avec le même nom.

---

## Vérificateur de Types

**Fichier :** `src/sema/checker.rs`

La structure `Checker` traverse l'AST et accumule des valeurs `TypeError`.

### État du Vérificateur

```rust
pub struct Checker<'a> {
    interner:          &'a Interner,
    loop_depth:        usize,
    functions:         HashMap<String, FunctionSignature>,
    fiber_context:     Option<Option<Type>>,  // None=hors fibre, Some(None)=void, Some(Some(T))=typée
    is_table_lambda:   bool,
    fiber_has_yield:   bool,
    in_yield_expr:     bool,
    last_expr_was_db_io: bool,
}
```

### Drapeaux de Contexte

| Champ | But |
|---|---|
| `loop_depth: usize` | Suit la profondeur d'imbrication de `while`/`for`. Zéro → `break`/`continue` sont des erreurs. Réinitialisé à 0 lors de l'entrée dans un corps de fibre. |
| `fiber_context: Option<Option<Type>>` | `None` = pas dans une fibre ; `Some(None)` = fibre void ; `Some(Some(T))` = fibre typée produisant `T` |
| `fiber_has_yield: bool` | Défini quand `yield` est rencontré. Sauvegardé/restauré par les définitions de fibres imbriquées. |
| `is_table_lambda: bool` | Défini à l'intérieur des prédicats `.where()` ; permet les noms de colonnes nus comme identifiants via `__row_tmp` |
| `in_yield_expr: bool` | Suit si nous sommes à l'intérieur d'une expression yield |
| `last_expr_was_db_io: bool` | Drapeau pour les opérations E/S de base de données |

### Passe de Pré-analyse

Avant de vérifier un corps de déclaration, le vérificateur effectue un **scan de déclaration anticipée** de tous les nœuds `FunctionDef` et `FiberDef` dans la **liste de déclarations actuelle** (non récursivement). Cela permet aux fonctions et fibres d'être appelées avant leur définition dans le fichier source (récursion mutuelle, appel avant déclaration).

Pour chaque fonction/fibre trouvée, le vérificateur enregistre :
- `FunctionSignature { params: Vec<Type>, return_type: Option<Type>, is_fiber: bool }` dans `self.functions`
- Une entrée dans la `SymbolTable` avec `Type::Unknown` (pour les fonctions) ou `Type::Fiber(...)` (pour les fibres)

Les fonctions de conversion intégrées `i`, `f`, `s`, `b` sont pré-enregistrées.

### Règles d'Inférence de Types

- Les types d'expression sont inférés de bas en haut depuis les littéraux et propagés à travers les opérateurs.
- `Type::Unknown` agit comme un joker — toute opération avec `Unknown` passe sans erreur.
- `Type::Json` est compatible avec tout type dans les assignations et comparaisons.
- Promotion numérique : `Int op Float → Float`.
- Un littéral de tableau vide `[]` hérite son type du contexte d'assignation si disponible.
- `Type::Table([])` (liste de colonnes vide) est compatible avec tout `Table(cols)`.

---

## Règles de Compatibilité des Types

La fonction `is_compatible(expected, actual) -> bool` :

| Règle | Description |
|---|---|
| L'un ou l'autre est `Unknown` | Toujours compatible |
| L'un ou l'autre est `Json` | Toujours compatible |
| `Int` ↔ `Float` | Mutuellement compatibles (promotion numérique) |
| `Int` ↔ `Date` | Compatible (les horodatages sont des entiers) |
| `Set(X)` ↔ `Array(inner)` | Compatible quand le type d'élément correspond |
| `Set(N)` ↔ `Set(Z)` | Compatible (les deux sont des ensembles à types entiers) |
| `Set(S)` ↔ `Set(C)` | Compatible (les deux sont des ensembles à types chaînes) |
| `Table([])` ↔ `Table(cols)` | Compatible quand l'une des listes de colonnes est vide |
| `Table(a)` ↔ `Table(b)` | Compatible quand les longueurs correspondent et les types de colonnes correspondent par paire |
| `Map(k1,v1)` ↔ `Map(k2,v2)` | Vérifié récursivement |
| `Fiber(None)` ↔ `Fiber(None)` | Les deux fibres void |
| `Fiber(Some(T))` ↔ `Fiber(Some(U))` | Compatible si T et U sont compatibles |

---

## Codes d'Erreur

| Code | Condition |
|---|---|
| `[S101] UndefinedVariable(name)` | Nom utilisé avant déclaration |
| `[S102] RedefinedVariable(name)` | Nom déclaré deux fois dans la même portée |
| `[S103] TypeMismatch { expected, actual }` | Le type de l'expression ne correspond pas au type attendu |
| `[S104] InvalidBinaryOp { op, left, right }` | Opérateur utilisé avec des types incompatibles |
| `[S105] ConstReassignment(name)` | Assignation à une variable `const` |
| `[S106] BreakOutsideLoop` | `break` en dehors d'une `while`/`for` |
| `[S107] ContinueOutsideLoop` | `continue` en dehors d'une `while`/`for` |
| `[S108] IndexAccessNotSupported(type)` | Indexation d'un type non supporté |
| `[S109] PropertyNotFound` | La propriété n'existe pas sur le type |
| `[S110] MethodNotFound` | La méthode n'existe pas sur le type |
| `[S111] InvalidArgumentCount` | Nombre incorrect d'arguments |
| `[S208] YieldOutsideFiber` | `yield` utilisé en dehors d'un corps de fibre |
| `[S209] FiberTypeMismatch` | `yield expr;` à l'intérieur d'une fibre void (devrait être `yield;`) |
| `[S210] ReturnTypeMismatchInFiber` | `return;` nu dans une fibre typée (valeur de retour manquante) |
| `[S211] CannotIterateOverVoidFiber` | Itération sur une fibre void |
| `[S212] CannotRunTypedFiber` | Appel de `.run()` sur une fibre typée |
| `[S301] WherePredicateNameCollision` | Le nom de la variable locale entre en conflit avec le nom d'une colonne dans `.where()` |
| `[S302] TableRowCountMismatch` | La ligne de la table a un nombre différent de colonnes que le schéma |
| `[D401] Rule violation` | `remove()` sans `.where()` |
| `Other(msg)` | Erreurs contextuelles diverses (nombre d'args, méthode inconnue, etc.) |

---

## Rapport d'Erreurs

Les valeurs `TypeError` portent un `Span { line, col, len }`. Après la vérification, `main.rs` passe chaque erreur à `Reporter::error()`, qui imprime :

```
ERROR: [S101] Undefined variable: foo
   12 | i: x = foo + 1;
              ~~~
```

La compilation s'arrête immédiatement après le rapport des erreurs — le bytecode n'est pas généré si des `TypeError` existent.

---

## Vérification des Appels de Fonctions

Pour les appels de fonctions nommées (`ExprKind::FunctionCall`) :
1. Cherche d'abord dans `self.functions` (pour les fonctions/fibres déclarées)
2. Si non trouvé, cherche dans la table des symboles (pour les valeurs de fonctions de première classe, appels dynamiques)
3. Si la signature résolue est une fibre, l'appel retourne `Type::Fiber(Some(ret))` (instanciation de fibre, pas d'appel direct)
4. Les arguments supplémentaires au-delà du nombre de paramètres déclarés sont vérifiés mais pas rejetés (tolérance variadique)
5. Les appels non résolus ajoutent une erreur `UndefinedVariable`

---

## Vérification de `.where()` sur les Tables

Lors de la vérification de `table.where(prédicat)` :
1. Une portée temporaire est ouverte et `__row_tmp: Table(cols)` est défini
2. `is_table_lambda` est défini à `true`
3. `collect_pred_idents()` collecte tous les noms `Identifier` utilisés dans le prédicat
4. Si un identifiant existe dans la portée externe **et** correspond à un nom de colonne → `S301 WherePredicateNameCollision`
5. Le prédicat est vérifié ; il doit retourner `Bool`
6. La portée est fermée et `is_table_lambda` est restauré