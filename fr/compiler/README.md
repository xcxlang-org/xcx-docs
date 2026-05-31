# Compilateur XCX — Documentation v3.1

> Compilateur du langage XCX implémenté en Rust. Pipeline multi-étapes : lexer → parseur → analyse sémantique → compilateur bytecode → machine virtuelle → JIT (Cranelift).

---

## Table des Matières

1. [Vue d'Ensemble de l'Architecture](#vue-densemble-de-larchitecture)
2. [Démarrage Rapide](#démarrage-rapide)
3. [Structure du Projet](#structure-du-projet)
4. [Pipeline de Compilation](#pipeline-de-compilation)
5. [Modules](#modules)
   - [Lexer (Scanner)](#lexer-scanner)
   - [Parseur (Pratt)](#parseur-pratt)
   - [Expandeur](#expandeur)
   - [Analyse Sémantique (Sema)](#analyse-sémantique-sema)
   - [Compilateur (Backend)](#compilateur-backend)
   - [Machine Virtuelle (VM)](#machine-virtuelle-vm)
   - [JIT (Cranelift)](#jit-cranelift)
6. [Décisions de Conception Clés](#décisions-de-conception-clés)
7. [Système de Diagnostic](#système-de-diagnostic)
8. [Sécurité](#sécurité)

---

## Vue d'Ensemble de l'Architecture

```
Code Source (.xcx)
        │
        ▼
  1. Lexer         src/lexer/scanner.rs       → flux de tokens
        │
        ▼
  2. Parseur        src/parser/pratt.rs        → AST brut (Program)
        │
        ▼
  3. Expandeur      src/parser/expander.rs     → AST étendu (include, alias)
        │
        ▼
  4. Sema          src/sema/checker.rs        → AST validé et annoté
        │
        ▼
  5. Compilateur    src/backend/mod.rs         → FunctionChunk + constantes + fonctions
        │
        ▼
  6. VM            src/backend/vm.rs          → exécution bytecode (basée sur registres)
        │  boucles chaudes → enregistrement de traces
        ▼
  7. JIT           src/backend/jit.rs         → code machine natif (Cranelift)
```

---

## Démarrage Rapide

```bash
# Démarrer le REPL
xcx

# Exécuter un fichier
xcx program.xcx

# Version
xcx --version

# Aide
xcx --help
```

**Dans le REPL :**

```
xcx> !help     # afficher l'aide
xcx> !clear    # effacer l'écran
xcx> !exit     # quitter
```

---

## Structure du Projet

```
src/
├── lexer/
│   ├── scanner.rs        # Scanner d'octets (&[u8])
│   └── token.rs          # TokenKind et Span
├── parser/
│   ├── pratt.rs          # Parseur Pratt (tokens → AST)
│   ├── expander.rs       # Résolution des include et préfixage des alias
│   └── ast.rs            # Définitions des nœuds AST (Expr, Stmt, Type, Program)
├── sema/
│   ├── checker.rs        # Vérification des types et résolution des variables
│   ├── symbol_table.rs   # Table des symboles hiérarchique
│   └── interner.rs       # Internement des chaînes (str → StringId)
├── backend/
│   ├── mod.rs            # Compilateur bytecode (AST → OpCode)
│   ├── vm.rs             # VM basée sur registres avec NaN-boxing + hooks JIT
│   ├── jit.rs            # Compilateur de traces Cranelift
│   └── repl.rs           # REPL interactif
└── diagnostic/
    └── report.rs         # Rapporteur d'erreurs avec coloration du code source
```

---

## Pipeline de Compilation

### 1. Lexer
Convertit les octets source en tokens. Opère sur `&[u8]` — aucune allocation de `Vec<char>`.

### 2. Parseur
Le parseur Pratt (Précédence d'Opérateurs Descendante) construit l'AST. Anticipation d'un seul token. Récupération d'erreur via `synchronize()`.

### 3. Expandeur
S'exécute **après** le parsage, **avant** l'analyse sémantique. Résout les directives `include` et préfixe les noms d'alias.

### 4. Analyse Sémantique
Vérifie les types, détecte les variables indéfinies, valide le contexte fibre/boucle. Collecte toutes les erreurs avant de générer le bytecode.

### 5. Compilateur
En deux passes. Passe 1 : enregistre les globaux/fonctions. Passe 2 : émet le bytecode avec des annotations Span.

### 6. VM
Machine virtuelle basée sur registres. Chaque frame possède un `Vec<Value>` plat indexé par des numéros de slots `u8`. Valeurs de 8 octets (NaN-boxing).

### 7. JIT
Compilation automatique des boucles chaudes en code natif via Cranelift. Transparent pour le développeur.

---

## Modules

Descriptions détaillées dans des fichiers séparés :

- [`docs/lexer.md`](docs/lexer.md) — Lexer / Scanner
- [`docs/parser.md`](docs/parser.md) — Parseur Pratt et AST
- [`docs/expander.md`](docs/expander.md) — Expandeur
- [`docs/sema.md`](docs/sema.md) — Analyse Sémantique
- [`docs/backend.md`](docs/backend.md) — Compilateur et VM
- [`docs/jit.md`](docs/jit.md) — JIT Cranelift
- [`docs/language.md`](docs/language.md) — Référence du Langage XCX

---

## Décisions de Conception Clés

### NaN-Boxing des valeurs
Chaque valeur est un seul `u64` (8 octets). Les bits de poids fort du NaN silencieux IEEE 754 servent d'étiquettes de type. Les 48 bits inférieurs transportent la charge utile. Aucune allocation sur le tas pour les scalaires, aucune surcharge d'étiquette dans les enums, aucune indirection de pointeur pour les primitives.

### VM basée sur registres
Au lieu d'une pile d'opérandes — un tableau `Vec<Value>` plat par frame. Les opcodes référencent directement les registres source/destination. Allocation simple de registres par pointeur de bosse dans le compilateur.

### JIT par Traçage (Cranelift)
Les sauts en arrière (arêtes de boucle) sont comptés par IP. Après avoir atteint un seuil (50 itérations), l'enregistrement de traces commence. La trace terminée est compilée en code natif par Cranelift. Les gardes de type (`GuardInt`, `GuardFloat`) gèrent la spécialisation de type ; une garde échouée déclenche un repli vers l'interpréteur.

### Internement des Chaînes
Tous les identifiants et littéraux de chaînes sont internés par l'`Interner` vers `StringId (u32)`. Élimine les allocations de chaînes répétitives et les comparaisons sur le tas dans tout le pipeline.

### Compilation en Deux Passes
La passe 1 (`register_globals_recursive`) assigne des indices à tous les globaux et fonctions avant d'émettre tout bytecode — permettant la récursion mutuelle et les appels avant déclaration.

---

## Système de Diagnostic

Le `Reporter` dans `src/diagnostic/report.rs` produit des messages d'erreur contextuels :

```
ERROR: [S101] Undefined variable: foo
   12 | i: x = foo + 1;
              ~~~
```

Chaque erreur contient :
- **Niveau** : ERROR ou HALT
- **Localisation** : numéro de ligne et de colonne
- **Coloration visuelle** : ligne source pertinente avec soulignement `~~~`

Les erreurs sémantiques (`TypeError`) sont collectées dans un `Vec` pendant la phase de vérification et signalées toutes en même temps avant la génération du bytecode.

---

## Sécurité

### Protection SSRF dans le Réseau
Cibles bloquées :
- URLs `file://`
- `169.254.x.x` (lien local / point de terminaison des métadonnées AWS)
- Plages privées : `10.x`, `192.168.x`, `172.16–31.x` (sauf localhost)

Appliqué à la fois à `HttpCall` et `HttpRequest`.

### Limite de Taille du Contenu HTTP
Dans `HttpServe`, le contenu de la réponse est vérifié après `into_string()`. S'il dépasse **10 Mo**, une erreur JSON 413 est renvoyée à la place du contenu réel.

### En-têtes CORS
Toutes les réponses `HttpServe` incluent automatiquement :
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

Les requêtes préliminaires `OPTIONS` reçoivent une réponse `204` sans appeler le gestionnaire.

### Chemins de Fichiers Sécurisés
Les opérations `store.*` valident les chemins — bloquant `..`, les chemins absolus et les lettres de lecteur Windows.