# Parseur XCX — v3.1

Le Parseur XCX transforme le flux de tokens en un Arbre Syntaxique Abstrait (AST) de haut niveau.

## Architecture : Analyse Pratt

XCX utilise un **Parseur Pratt** (Précédence d'Opérateurs Descendante).

- **Fichier** : `src/parser/pratt.rs`
- **Anticipation** : Un token (`current` + `peek`), avancé manuellement avec `advance()`.
- **Récupération d'Erreur** : Sur une erreur de syntaxe, `synchronize()` saute des tokens jusqu'au prochain point-virgule ou un mot-clé connu démarrant une déclaration (`func`, `fiber`, `if`, `for`, `const`, `return`, `>!`, etc.).

La structure `Parser` emprunte la chaîne source pour la durée de vie `'a`, et `Scanner<'a>` est paramétrisé par la même durée de vie, reflétant le scanner basé sur des tranches d'octets.

### Niveaux de Précédence (du plus bas au plus haut)

| Niveau | Opérateurs |
|---|---|
| `Lowest` | — |
| `Lambda` | `->` |
| `Assignment` | `=` |
| `LogicalOr` | `OR`, `\|\|` |
| `LogicalAnd` | `AND`, `&&` |
| `Equals` | `==`, `!=` |
| `LessGreater` | `>`, `<`, `>=`, `<=`, `HAS` |
| `Sum` | `+`, `-`, `++` |
| `SetOp` | `UNION`, `INTERSECTION`, `DIFFERENCE`, `SYMMETRIC_DIFFERENCE` |
| `Product` | `*`, `/`, `%` |
| `Power` | `^` |
| `Prefix` | `-x` |
| `Call` | `.`, `[` |

## Dispatch des Déclarations

`parse_statement_internal()` dispatch sur le token actuel :

- **Mots-clés de type** (`i`, `f`, `s`, `b`, `array`, `set`, `map`, `date`, `table`, `json`) → `parse_var_decl()`, ou `parse_assignment()` si suivi de `=`
- **`const`** → `parse_var_decl()` avec `is_const = true`
- **`var`** (identifiant) → déclaration de variable avec inférence de type
- **`>!`** → `parse_print_stmt()`
- **`>?`** → `parse_input_stmt()`
- **`halt`** → `parse_halt_stmt()`
- **`if`** → `parse_if_statement()`
- **`while`** → `parse_while_statement()`
- **`for`** → `parse_for_statement()`
- **`break`** / **`continue`** → `parse_break_statement()` / `parse_continue_statement()`
- **`func`** → `parse_func_def()`
- **`fiber`** → `parse_fiber_statement()` (dispatch vers def ou decl selon peek)
- **`return`** → `parse_return_stmt()`
- **`yield`** → `parse_yield_stmt()` (gère `yield expr`, `yield from expr` et `yield;`)
- **`@wait`** → `parse_wait_stmt()`
- **`serve`** → `parse_serve_stmt()`
- **`net`** → `parse_net_stmt()`
- **`include`** → `parse_include_stmt()`
- **Identifiant + `=`** → `parse_assignment()`
- **Identifiant + `(`** → `parse_func_call_stmt()`
- **Tout le reste** → `parse_expr_stmt()`

## Styles de Définition de Fonction

XCX supporte deux styles syntaxiquement différents pour définir des fonctions :

**Style accolades** (C-like) :
```xcx
func name(i: x, s: y -> i) {
    return x + 1;
}
```

**Style XCX** (bloc de mots-clés) :
```xcx
func:i: name(i: x, s: y) do;
    return x + 1;
end;
```

Les deux produisent des nœuds AST `StmtKind::FunctionDef` identiques. Le type de retour dans le style accolades est déclaré avec `-> type` à l'intérieur de la liste des paramètres ou après `)`.

## Déclarations de Fibres

`parse_fiber_statement()` regarde `peek` pour décider :
- `peek == Colon` → `parse_fiber_decl()` (instanciation : `fiber:T: varname = fiberDef(args);`)
- sinon → `parse_fiber_def()` (définition : `fiber name(params) { body }`)

`parse_fiber_decl()` gère également le cas où, après avoir analysé le type et le nom, le token actuel est `(` — dans ce cas il pivote vers `finish_fiber_def()` (une définition avec une annotation de type `fiber:` au début).

## Constructions Clés Analysées

- **Déclarations de variables** : `i: name = expr;`, `const s: NAME = expr;`, `var name = expr;`
- **Flux de contrôle** : `if (cond) then; ... elseif (cond) then; ... else; ... end;`
- **Boucle while** : `while (cond) do; ... end;`
- **Boucle for** : `for x in expr do; ... end;` et `for x in start to end @step n do; ... end;`
- **Fonctions** : `func` (deux styles, voir ci-dessus)
- **Fibres** : `fiber name(params) { body }` et `fiber:T: varname = fiberName(args);`
- **Yield** : `yield expr;`, `yield from expr;`, `yield;`
- **HTTP** : `serve: name { port=..., routes=... };`, `net.get(url)`, `net.request { ... } as resp;`, `net.respond(status, body);`
- **Collections** : Tableau `[a, b, c]`, Ensemble `set:N { 1,,10 }`, Map `[k :: v, ...]`, Table `table { columns=[...] rows=[...] }`
- **Blocs bruts** : `<<<...>>>` pour JSON/chaînes inline
- **Include** : `include "path";` ou `include "path" as alias;`
- **E/S** : `>! expr;` (impression), `>? varname;` (entrée)
- **Halt** : `halt.alert >! msg;`, `halt.error >! msg;`, `halt.fatal >! msg;`
- **Attente** : `@wait(ms);` ou `@wait ms;`
- **Littéraux de date** : `date("2024-01-01")` ou `date("01/01/2024", "DD/MM/YYYY")`

## Analyse des Expressions

`parse_expression(precedence)` appelle `parse_prefix()` pour le côté gauche, puis boucle en appelant `parse_infix(left)` tant que la précédence du token d'anticipation dépasse le minimum actuel.

Parseurs de préfixe clés :
- **Identifiants** : Si suivi de `(`, analysé comme `FunctionCall` ; sinon comme `Identifier`.
- **Littéraux** : `IntLiteral`, `FloatLiteral`, `StringLiteral`, `True`, `False`
- **Moins unaire** : Analysé comme `Binary { left: IntLiteral(0), op: Minus, right }` (pas de `Unary::Neg` séparé)
- **`not` / `!`** : `Unary { op: Not/Bang, right }`
- **Groupes `(`...`)`** : Expression unique → dépaquetée ; plusieurs séparées par des virgules → `Tuple`
- **`[`...`]`** : Si le premier élément est suivi de `::`, analysé comme `MapLiteral` ; sinon `ArrayLiteral`
- **`{`...`}`** : Analysé comme `ArrayOrSetLiteral` (type résolu au moment sémantique ou de la compilation)
- **`set:N { }` etc.** : `SetLiteral` explicite avec `SetType` connu
- **`map { schema=[...] data=[...] }`** : `MapLiteral` explicite
- **`table { columns=[...] rows=[...] }`** : `TableLiteral`
- **`random.choice from expr`** : `RandomChoice`
- **`date(...)`** : `DateLiteral`
- **`net.get/post/put/delete/patch(...)` etc.** : `NetCall` ou `NetRespond`
- **`<<<...>>>`** : `RawBlock`
- **`.terminal!cmd`** : `TerminalCommand`

Parseurs d'infixe clés :
- **`.`** : `parse_dot_infix` — produit `MethodCall` si suivi de `(`, sinon `MemberAccess` ; gère également l'accès à l'index `.[key]`
- **`[`** : `parse_index_infix` → `Index`
- **`->`** : `parse_lambda_infix` → `Lambda`
- **Tous les opérateurs binaires** : `Binary { left, op, right }`

## Post-Traitement de `parse_expr_stmt()`

Après avoir analysé une déclaration d'expression complète, `parse_expr_stmt()` vérifie si le résultat est un `MethodCall` :
- Nom de méthode `bind` avec 2 args et le deuxième arg est un `Identifier` → réécrit comme `StmtKind::JsonBind`
- Nom de méthode `inject` avec 2 args → réécrit comme `StmtKind::JsonInject`

Cela permet la syntaxe sucrée `json.bind("path", target);` et `json.inject(mapping, table);` au niveau de la déclaration.

## Expandeur (`src/parser/expander.rs`)

L'Expandeur s'exécute **après** le parsage, **avant** l'analyse sémantique. C'est une passe de réécriture d'arbre séparée.

### Responsabilités

**Résolution des includes** : `include "file.xcx";` est remplacé par l'AST inliné de ce fichier. Les dépendances circulaires sont détectées via `visiting_files: HashSet<PathBuf>`. Les fichiers sont dédupliqués via `included_files: HashSet<PathBuf>` (chaque fichier inclus une seule fois sauf avec alias).

**Préfixage des alias** : `include "math.xcx" as math;` fait que tous les noms de niveau supérieur de ce fichier sont renommés en `math.nom`. Les sites d'appel (`math.sin(x)`) sont réécrits de `MethodCall` vers `FunctionCall { name: "math.sin" }` par `expand_expr_inplace`. Les fonctions `prefix_program()` / `prefix_stmt_impl()` / `prefix_expr_impl()` parcourent l'ensemble du sous-AST en renommant toutes les références aux symboles de niveau supérieur.

**Préfixage des noms de fibres** : Les références `FiberDecl::fiber_name` sont également préfixées pour que les instanciations de fibres renommées se résolvent correctement après le préfixage.

**Préfixage de YieldFrom** : Les expressions `StmtKind::YieldFrom` sont parcourues pour que les appels de constructeurs de fibres à l'intérieur de `yield from` soient également renommés.

**Noms protégés** (jamais préfixés) : `json`, `date`, `store`, `halt`, `terminal`, `net`, `env`, `crypto`, `EMPTY`, `math`, `random`, `i`, `f`, `s`, `b`, `from`, `main`.

**Ordre de recherche des chemins d'include** :
1. Relatif au répertoire du fichier actuel
2. Dans le répertoire `lib/` (relatif au répertoire courant, puis en remontant depuis le chemin de l'exécutable)

## Définitions de l'AST (`src/parser/ast.rs`)

### `Expr` — Nœuds d'Expression

| Variante | Description |
|---|---|
| `IntLiteral(i64)` | Constante entière |
| `FloatLiteral(f64)` | Constante flottante |
| `StringLiteral(StringId)` | Chaîne internée |
| `BoolLiteral(bool)` | `true` / `false` |
| `Identifier(StringId)` | Nom de variable ou de fonction |
| `Binary { left, op, right }` | Opération binaire |
| `Unary { op, right }` | Opération unaire (`not`, `!`) |
| `FunctionCall { name, args }` | Appel de fonction par nom interné |
| `MethodCall { receiver, method, args }` | Appel par point sur une valeur |
| `MemberAccess { receiver, member }` | Accès par point sans appel |
| `Index { receiver, index }` | Index par crochet `a[i]` |
| `Lambda { params, return_type, body }` | Lambda flèche `x -> expr` |
| `ArrayLiteral { elements }` | `[a, b, c]` explicite |
| `ArrayOrSetLiteral { elements }` | `{a, b, c}` ambigu — résolu plus tard |
| `SetLiteral { set_type, elements, range }` | Ensemble typé avec plage optionnelle |
| `MapLiteral { key_type, value_type, elements }` | Littéral de map |
| `TableLiteral { columns, rows }` | Littéral de table |
| `DateLiteral { date_string, format }` | `date("2024-01-01")` |
| `Tuple(Vec<Expr>)` | Liste séparée par des virgules entre parenthèses |
| `NetCall { method, url, body }` | Expression d'appel HTTP |
| `NetRespond { status, body, headers }` | Expression de réponse HTTP |
| `RawBlock(StringId)` | Contenu brut `<<<...>>>` |
| `TerminalCommand(cmd, arg)` | `.terminal !cmd` |
| `RandomChoice { set }` | `random.choice from set` |

### `Stmt` — Nœuds de Déclaration

Variantes clés : `VarDecl`, `Assign`, `Print`, `Input`, `If`, `While`, `For`, `Break`, `Continue`, `FunctionDef`, `FiberDef`, `FiberDecl`, `Return`, `Yield`, `YieldFrom`, `YieldVoid`, `Include`, `Serve`, `NetRequestStmt`, `JsonBind`, `JsonInject`, `Halt`, `Wait`, `ExprStmt`, `FunctionCallStmt`.

### `Type` — Système de Types

`Int`, `Float`, `String`, `Bool`, `Date`, `Json`, `Array(Box<Type>)`, `Set(SetType)`, `Map(Box<Type>, Box<Type>)`, `Table(Vec<ColumnDef>)`, `Fiber(Option<Box<Type>>)`, `Builtin(StringId)`, `Unknown`.

Variantes de `SetType` : `N` (Naturels), `Z` (Entiers), `Q` (Rationnels/Flottants), `S` (Chaînes), `C` (Char/Chaînes), `B` (Booléens).

### `ForIterType`

`Range` (numérique `start to end`), `Array`, `Set`, `Fiber` — défini par le vérificateur de types et utilisé par le compilateur pour émettre le bon motif de boucle.

### `ColumnDef`

```rust
pub struct ColumnDef {
    pub name:    StringId,
    pub ty:      Type,
    pub is_auto: bool,    // les colonnes @auto sont auto-incrémentées à l'insertion
}
```

## Internement des Chaînes

Toutes les valeurs de chaînes (identifiants, littéraux de chaînes, noms de méthodes) sont internées via `Interner` en `StringId (u32)`. L'internement est créé dans le parseur et passé à travers toutes les phases suivantes. Cela signifie que le vérificateur, le compilateur et la VM utilisent tous des IDs numériques pour les comparaisons de noms au lieu de comparaisons de `String`.