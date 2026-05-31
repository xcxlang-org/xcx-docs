# Analyse Sémantique XCX (Sema) — v3.1

La phase Sema valide l'AST pour vérifier la cohérence logique et des types avant la génération du bytecode. Elle se compose de deux éléments : la **Table des Symboles** et le **Vérificateur de Types**.

## Internement de Chaînes (`src/sema/interner.rs`)

`Interner` fait correspondre `&str → StringId(u32)`. C'est la source de vérité unique pour toutes les identités de chaînes dans le compilateur. Créé lors de la phase de lexing/parsing, puis transmis par référence au vérificateur et au compilateur.

```
Interner::intern("foo") → StringId(42)
Interner::lookup(StringId(42)) → "foo"
```

## Table des Symboles (`src/sema/symbol_table.rs`)

La `SymbolTable` gère les liaisons de variables à travers des portées imbriquées via une **chaîne à pointeurs parents**.

### Structure

```rust
pub struct SymbolTable<'a> {
    parent: Option<&'a SymbolTable<'a>>,  // référence à la portée englobante
    scopes: Vec<HashMap<String, Type>>,   // pile de frames de portée (locale à cette table)
    consts: Vec<HashSet<String>>,         // quels noms sont const, par frame de portée
}
```

### Création d'une Portée Enfant

Lors de l'entrée dans le corps d'une fonction ou d'une fibre, une nouvelle `SymbolTable` est créée avec une référence à la table englobante, sans la cloner :

```rust
let mut func_symbols = SymbolTable::new_with_parent(symbols);
```

C'est une opération O(1) — une seule `HashMap` vide est allouée pour la nouvelle frame. La recherche remonte la chaîne de parents au besoin.

### Cycle de Vie des Portées

- `enter_scope()` / `exit_scope()` empilent/dépilent les frames locales dans une table — utilisées pour les corps de `if`, `while`, `for`.
- `new_with_parent(parent)` crée une table enfant liée au parent — utilisée pour les corps de fonctions et de fibres.
- `define(name, ty, is_const)` écrit toujours dans la portée la plus **intérieure** (courante) de la table actuelle.
- `lookup(name)` parcourt les portées locales de l'intérieur vers l'extérieur, puis suit la chaîne de pointeurs `parent`.
- `has_in_current_scope(name)` vérifie uniquement la frame la plus intérieure de la table courante — utilisée pour détecter les redéfinitions dans le même bloc.
- `is_const(name)` trouve la portée qui possède `name`, puis vérifie `consts[scope_index]`.

### Important : Pas d'Ombrage de Variables

XCX **ne prend pas en charge** l'ombrage de variables. Définir une variable qui existe déjà dans la **portée courante** génère `RedefinedVariable`. Les variables des portées parentes sont accessibles mais ne peuvent pas être redéclarées dans une portée enfant avec le même nom.

## Vérificateur de Types (`src/sema/checker.rs`)

La structure `Checker` parcourt l'AST et accumule des valeurs `TypeError`. Le programme n'est compilé que si le `Vec<TypeError>` résultant est vide.

### État du Vérificateur

```rust
pub struct Checker<'a> {
    interner:       &'a Interner,
    loop_depth:     usize,
    functions:      HashMap<String, FunctionSignature>,
    fiber_context:  Option<Option<Type>>,
    is_table_lambda: bool,
    fiber_has_yield: bool,
}
```

### Passe de Pré-scan (`pre_scan_stmts`)

Avant de vérifier tout corps de déclaration, le vérificateur effectue un **scan de déclaration anticipée** de tous les nœuds `FunctionDef` et `FiberDef` de la **liste de déclarations courante** (non récursif). Cela permet d'appeler des fonctions et des fibres avant leur définition dans le fichier source (récursion mutuelle, appel avant déclaration).

Pour chaque fonction/fibre trouvée, le vérificateur enregistre :
- Une `FunctionSignature { params: Vec<Type>, return_type: Option<Type>, is_fiber: bool }` dans `self.functions`
- Une entrée dans la `SymbolTable` avec `Type::Unknown` (pour les fonctions) ou `Type::Fiber(...)` (pour les fibres)

Les fonctions de transtypage intégrées `i`, `f`, `s`, `b` sont pré-enregistrées avec le paramètre `Type::Unknown` et les types de retour correspondants.

### Drapeaux de Contexte

| Champ | Rôle |
|---|---|
| `loop_depth: usize` | Suit la profondeur d'imbrication des `while`/`for`. Zéro → `break`/`continue` sont des erreurs. Remis à 0 lors de l'entrée dans le corps d'une fibre. |
| `fiber_context: Option<Option<Type>>` | `None` = pas dans une fibre ; `Some(None)` = fibre void ; `Some(Some(T))` = fibre typée produisant `T`. |
| `fiber_has_yield: bool` | Activé quand un `yield` est rencontré. Sauvegardé/restauré lors des définitions de fibres imbriquées. |
| `is_table_lambda: bool` | Activé dans les prédicats `.where()` ; autorise les noms de colonnes nus comme identifiants en les recherchant dans `__row_tmp`. |

### Règles d'Inférence de Types

- Les types d'expressions sont inférés du bas vers le haut à partir des littéraux et se propagent via les opérateurs.
- `Type::Unknown` agit comme un joker — toute opération impliquant `Unknown` passe sans erreur.
- `Type::Json` est compatible avec n'importe quel type dans les affectations et les comparaisons.
- Promotion numérique : `Int op Float → Float`.
- Le littéral tableau vide `[]` hérite son type du contexte d'affectation si disponible.
- `Type::Table([])` (liste de colonnes vide) est compatible avec tout `Table(cols)` — les informations de colonnes sont propagées en retour dans le type déclaré de la variable après inférence.

### `is_compatible(expected, actual) -> bool`

Règles de compatibilité principales :

| Règle | Description |
|---|---|
| L'un ou l'autre est `Unknown` | Toujours compatible |
| L'un ou l'autre est `Json` | Toujours compatible |
| `Int` ↔ `Float` | Mutuellement compatibles (promotion numérique) |
| `Int` ↔ `Date` | Compatibles (les horodatages sont des entiers) |
| `Set(X)` ↔ `Array(inner)` | Compatible quand le type de l'élément interne correspond |
| `Set(N)` ↔ `Set(Z)` | Compatible (les deux sont des ensembles à type entier) |
| `Set(S)` ↔ `Set(C)` | Compatible (les deux sont des ensembles à type chaîne) |
| `Table([])` ↔ `Table(cols)` | Compatible quand l'une des listes de colonnes est vide |
| `Table(a)` ↔ `Table(b)` | Compatible quand les longueurs correspondent et que tous les types de colonnes sont pairwise compatibles |
| `Map(k1,v1)` ↔ `Map(k2,v2)` | Vérifié récursivement |
| `Fiber(None)` ↔ `Fiber(None)` | Les deux sont des fibres void |
| `Fiber(Some(T))` ↔ `Fiber(Some(U))` | Compatible ssi T et U sont compatibles |

### Vérification des Appels de Fonctions

Pour les appels de fonctions nommées (`ExprKind::FunctionCall`) :
1. Recherche d'abord dans `self.functions` (pour les fonctions/fibres déclarées).
2. Si non trouvé, recherche dans la table des symboles (pour les valeurs de fonctions de premier ordre, appels dynamiques).
3. Si la signature résolue est une fibre, l'appel retourne `Type::Fiber(Some(ret))` (instanciation de fibre, pas invocation directe).
4. Les arguments supplémentaires au-delà du nombre de paramètres déclarés sont vérifiés mais non rejetés (tolérance variadique).
5. Les appels non résolus génèrent `UndefinedVariable`.

Pour les appels de fonctions au niveau des déclarations (`StmtKind::FunctionCallStmt`) :
- Nombre d'arguments incorrect → `Other("Function expects N arguments, got M")`.
- Nom inconnu absent de la table des symboles → `UndefinedVariable`.

### Vérification des Déclarations de Fibres (`FiberDecl`)

1. Recherche `fiber_name` dans `self.functions` (en priorité) ou dans la table des symboles.
2. Vérifie que la signature résolue a `is_fiber: true`.
3. Vérifie les types des arguments par rapport aux paramètres.
4. Définit la nouvelle variable avec `Type::Fiber(inner_type)`.

### Vérification de `.where()` sur les Tables

Lors de la vérification de `table.where(predicate)` :
1. Une portée temporaire est ouverte et `__row_tmp: Table(cols)` est défini.
2. `is_table_lambda` est mis à `true`.
3. `collect_pred_idents()` collecte tous les noms `Identifier` utilisés dans le prédicat.
4. Si un identifiant existe à la fois dans la portée externe **et** correspond à un nom de colonne → `S301 WherePredicateNameCollision`.
5. Le prédicat est vérifié par le système de types ; il doit retourner `Bool`.
6. La portée est fermée et `is_table_lambda` est restauré.

### Vérification de `Table.join()`

Le vérificateur fusionne les définitions de colonnes : colonnes de la table gauche plus colonnes de la table droite non déjà présentes (par égalité de `StringId`). En cas de conflit de nom de colonne, la colonne de la table droite est conservée. Le type résultant est `Type::Table(combined_cols)`.

### Vérification des Boucles `for`

Le champ `iter_type` de `ForIterType` est muté pendant la vérification en fonction du type inféré de `start` :
- `Type::Array(_)` → définit `ForIterType::Array`, la variable de boucle reçoit le type interne
- `Type::Set(st)` → définit `ForIterType::Set`, la variable de boucle reçoit le type d'élément de l'ensemble
- `Type::Table(cols)` → traité comme `ForIterType::Array`, la variable de boucle reçoit `Type::Table(cols)`
- `Type::Fiber(inner)` → définit `ForIterType::Fiber`, la variable de boucle reçoit le type de valeur produite par la fibre
- `Type::Int` (avec le mot-clé `to`) → reste `ForIterType::Range`, la variable de boucle est `Int`

---

## Codes d'Erreur Validés

| Code | Condition |
|---|---|
| `UndefinedVariable(name)` | Nom utilisé avant sa déclaration |
| `RedefinedVariable(name)` | Nom déclaré deux fois dans la même portée |
| `ConstReassignment(name)` | Affectation à une variable `const` |
| `TypeMismatch { expected, actual }` | Le type de l'expression ne correspond pas au type attendu |
| `InvalidBinaryOp { op, left, right }` | Opérateur utilisé avec des types incompatibles |
| `BreakOutsideLoop` | `break` hors d'un `while`/`for` |
| `ContinueOutsideLoop` | `continue` hors d'un `while`/`for` |
| `[S208] YieldOutsideFiber` | `yield` utilisé hors de tout corps de fibre |
| `[S209] FiberTypeMismatch` | `yield expr;` à l'intérieur d'une fibre void (devrait être `yield;`) |
| `[S210] ReturnTypeMismatchInFiber` | `return;` nu dans une fibre typée (valeur de retour manquante) |
| `[S301] WherePredicateNameCollision` | Un nom de variable locale entre en conflit avec un nom de colonne dans `.where()` |
| `Other(msg)` | Diverses erreurs contextuelles (nombre d'arguments, méthode inconnue, itération sur fibre void, etc.) |

---

## Rapport d'Erreurs

Les valeurs `TypeError` portent un `Span { line, col, len }`. Après la vérification, `main.rs` passe chaque erreur à `Reporter::error()`, qui affiche :
1. `ERROR: <message>`
2. La ligne source concernée avec son numéro de ligne
3. Un soulignement `~~~` débutant à `col` avec la longueur `len`

La compilation s'arrête immédiatement après le rapport d'erreurs — aucun bytecode n'est généré si un `TypeError` existe.