# Architecture du Compilateur XCX — v3.1

Le compilateur XCX est implémenté en Rust et suit une architecture de pipeline multi-étapes.

## Pipeline de Compilation

```
Code Source
    │
    ▼
1. Lexer (Scanner)        — src/lexer/scanner.rs
    │  Produit : Flux de tokens
    ▼
2. Parseur (Pratt)         — src/parser/pratt.rs
    │  Produit : AST brut (Program)
    ▼
3. Expandeur               — src/parser/expander.rs
    │  Produit : AST étendu (directives include résolues, alias préfixés)
    ▼
4. Vérificateur de Types (Sema)    — src/sema/checker.rs
    │  Produit : AST validé et annoté
    ▼
5. Compilateur (Backend)     — src/backend/mod.rs
    │  Produit : FunctionChunk (main) + Arc<Vec<Value>> constantes + Arc<Vec<FunctionChunk>> fonctions
    ▼
6. VM                     — src/backend/vm.rs
    │  Exécute le bytecode basé sur registres
    │  Boucles chaudes détectées → Enregistrement de traces commence
    ▼
7. JIT (Cranelift)        — src/backend/jit.rs
       Compile les traces enregistrées en code machine natif
```

> **Remarque** : L'Expandeur fait partie du module `src/parser/` mais s'exécute comme une phase distincte post-parsage, avant l'analyse sémantique. Le JIT est une couche d'accélération optionnelle qui s'active automatiquement pour les boucles chaudes — l'exécution du bytecode continue sans interruption si une trace n'est pas encore compilée.

## Structure du Projet

```
src/
├── lexer/
│   ├── scanner.rs      # Scanner au niveau octet (&[u8])
│   └── token.rs        # Définitions de TokenKind et Span
├── parser/
│   ├── pratt.rs        # Parseur Pratt (flux de tokens → AST)
│   ├── expander.rs     # Résolution des includes et préfixage des alias
│   └── ast.rs          # Définitions des nœuds AST (Expr, Stmt, Type, Program)
├── sema/
│   ├── checker.rs      # Vérificateur de types et résolveur de variables
│   ├── symbol_table.rs # Table de symboles/portées hiérarchique avec chaîne de pointeurs parents
│   └── interner.rs     # Internement de chaînes (str → StringId)
├── backend/
│   ├── mod.rs          # Compilateur bytecode (AST → OpCode) avec allocateur de registres
│   ├── vm.rs           # VM basée sur registres avec valeurs NaN-boxées + hooks JIT de traçage
│   ├── jit.rs          # Compilateur de traces basé sur Cranelift
│   └── repl.rs         # REPL interactif
└── diagnostic/
    └── report.rs       # Rapporteur d'erreurs avec coloration du code source
```

## Système de Diagnostic

Le compilateur utilise une structure `Reporter` pour produire des messages d'erreur contextuels. Chaque erreur inclut :
- **Niveau** : variante ERROR ou HALT
- **Localisation** : numéro de ligne et de colonne
- **Coloration visuelle** : la ligne source pertinente avec un soulignement `~~~`

Les erreurs sémantiques (`TypeError`) sont collectées dans un `Vec` pendant la phase de vérification et signalées toutes en même temps avant le début de la génération du bytecode. Si des erreurs existent, la compilation s'arrête à ce stade.

Les erreurs d'exécution produites par la VM incluent des informations de localisation dans le code source (ligne et colonne) dérivées de la table `spans` stockée avec le bytecode de chaque `FunctionChunk`.

## Décisions de Conception Clés

### Valeurs NaN-Boxées
Toutes les valeurs sont représentées comme un seul `u64` en utilisant le NaN-boxing. Les bits de poids fort d'un NaN silencieux IEEE 754 sont utilisés comme étiquettes de type, et les 48 bits inférieurs portent la charge utile (entier, booléen ou pointeur). Les flottants sont stockés tels quels. Cela signifie que chaque `Value` fait exactement 8 octets — aucune allocation sur le tas pour les scalaires, aucune surcharge d'étiquette sur les enums, aucune indirection de pointeur pour les primitives. Voir `src/backend/vm.rs` pour la disposition complète des étiquettes.

### VM Basée sur Registres
La VM est **basée sur registres**, non sur une pile. Chaque frame de fonction possède un `Vec<Value>` plat indexé par numéro de slot. Les opcodes référencent directement les registres source et destination (`dst`, `src1`, `src2`). Le `FunctionCompiler` du compilateur maintient un simple allocateur par bosse (`push_reg()` / `pop_reg()`) pour les registres temporaires. Les locaux et les temporaires vivent dans le même tableau plat — il n'y a pas de pile d'opérandes séparée.

### JIT par Traçage (Cranelift)
Les sauts en arrière chauds sont détectés par un compteur `hot_counts: Vec<usize>` par frame. Quand une arête de retour de boucle atteint le seuil (50 itérations), une phase d'enregistrement de trace commence. L'`Executor` enregistre des variantes `TraceOp` — versions typées et spécialisées des opcodes de l'interpréteur — jusqu'à ce que la trace se ferme (retour de boucle à l'IP de départ). La `Trace` complète est compilée en code natif via Cranelift et mise en cache dans `trace_cache`. Les itérations suivantes de cette boucle contournent entièrement l'interpréteur. Les gardes (`GuardInt`, `GuardFloat`, `GuardTrue`, `GuardFalse`) dans la trace gèrent la spécialisation de type ; une garde échouée provoque une sortie latérale vers l'interpréteur à l'IP correct.

### Internement des Chaînes
Tous les identifiants et littéraux de chaînes sont internés via `Interner` en `StringId` (u32). Cela évite les allocations de chaînes répétées et les comparaisons sur le tas dans tout le pipeline.

### Déduplication des Constantes
Le compilateur maintient un `string_constants: HashMap<String, usize>` qui assure que chaque valeur de chaîne unique est stockée une seule fois dans la table des constantes. Particulièrement efficace pour les noms de méthodes intégrés émis fréquemment pendant la compilation.

### Dispatch de Méthodes via Enum
Les appels de méthodes intégrées sont compilés en `OpCode::MethodCall { kind: MethodKind, base, arg_count }` où `MethodKind` est un enum `Copy` couvrant ~50 méthodes intégrées. Les noms de méthodes inconnus ou dynamiques (p.ex., accès à un champ JSON) utilisent le chemin séparé `OpCode::MethodCallCustom { method_name_idx, base, arg_count }`. Cela élimine les comparaisons de chaînes dans la boucle de dispatch de la VM.

### Compilation en Deux Passes
Le backend effectue une première passe (`register_globals_recursive`) pour assigner des indices à tous les globaux, fonctions et fibres avant d'émettre tout bytecode.

### Bytecode Annoté par Span
Chaque opcode émis est associé à un `Span` de l'AST source. `FunctionChunk` stocke `spans: Arc<Vec<Span>>` avec `bytecode: Arc<Vec<OpCode>>`. La VM utilise ceci pour produire des messages d'erreur d'exécution précis à la ligne.

### Fibre-comme-Coroutine
Les fibres ne sont pas des threads OS. Chaque `FiberState` stocke son propre `ip` et `locals`, repris coopérativement par la VM. À l'arrêt, les locaux sont déplacés vers `FiberState` ; à la reprise, ils sont déplacés à nouveau. Ceci est sans allocation au-delà de la création initiale de `Vec`.

### Représentation des Valeurs Thread-Safe
Toutes les collections mutables partagées utilisent `Arc<RwLock<T>>` (via `parking_lot`). Le bytecode et les spans de `FunctionChunk` sont enveloppés dans `Arc<Vec<...>>` pour pouvoir être partagés entre les threads de workers HTTP sans copie. `Value` est `Copy` + `Send` + `Sync`.

### Arrêt Gracieux HTTP
Un `AtomicBool` global (`SHUTDOWN`) est défini par un gestionnaire de signal Ctrl+C (via la crate `ctrlc`). Tous les threads de workers HTTP et la boucle de dispatch principale interrogent ce drapeau et sortent proprement avant la fin du processus.

### Table des Symboles en Chaîne de Portées
La `SymbolTable` utilise une chaîne liée par pointeurs parents au lieu du clonage profond. L'entrée dans une portée de fonction crée une nouvelle `SymbolTable` avec une référence au parent — la recherche remonte la chaîne. L'entrée dans une portée de fonction est O(1) au lieu de O(n).

### Scanner au Niveau Octet
Le lexer opère sur `&[u8]` (une référence aux octets source d'origine) sans allouer un `Vec<char>`. La gestion de l'Unicode est faite uniquement là où c'est nécessaire. La détection de commentaires utilise `slice.starts_with(b"---")`.