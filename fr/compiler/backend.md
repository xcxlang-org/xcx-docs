# Backend XCX — Compilateur et VM

> **Fichiers :** `src/backend/mod.rs`, `src/backend/vm.rs`

---

## Table des Matières

1. [Compilateur Bytecode](#compilateur-bytecode)
2. [Représentation des Valeurs — NaN-Boxing](#représentation-des-valeurs--nan-boxing)
3. [Jeu d'Instructions (OpCodes)](#jeu-dinstructions-opcodes)
4. [Architecture de la VM](#architecture-de-la-vm)
5. [Modèle d'Exécution des Fibres](#modèle-dexécution-des-fibres)
6. [Serveur HTTP](#serveur-http)
7. [Gestion de la Mémoire](#gestion-de-la-mémoire)
8. [Optimisations de Boucles](#optimisations-de-boucles)

---

## Compilateur Bytecode

**Fichier :** `src/backend/mod.rs`

### Allocation de Registres

`FunctionCompiler` suit le prochain registre disponible via `next_local: usize` :

```rust
pub fn push_reg(&mut self) -> u8 {
    let r = self.next_local as u8;
    self.next_local += 1;
    if self.next_local > self.max_locals_used {
        self.max_locals_used = self.next_local;
    }
    r
}

pub fn pop_reg(&mut self) {
    self.next_local -= 1;
}
```

Les variables locales (nommées) sont assignées à un slot via `define_local(id, slot)` et stockées dans `scopes: Vec<HashMap<StringId, usize>>`. Les variables temporaires utilisent `push_reg()`/`pop_reg()` — elles sont réutilisées lorsque le résultat d'une expression est consommé.

`max_locals_used` est enregistré pour que `FunctionChunk::max_locals` puisse pré-allouer un vecteur de locaux de taille exacte lors de l'appel de la fonction.

### Déduplication des Constantes

`CompileContext::add_constant` déduplique les constantes de chaînes via `string_constants: HashMap<String, usize>`. Les constantes de chaînes dupliquées (p.ex., `"insert"` apparaissant plusieurs fois comme argument de nom de méthode) réutilisent le même slot de table de constantes.

### Compilation en Deux Passes

**Passe 1** — `register_globals_recursive` :
- Assigne un indice de slot à chaque variable globale et instance de déclaration de fibre
- Assigne un indice de fonction à chaque fonction/fibre
- Pré-alloue des slots `FunctionChunk` vides dans `functions: Vec<FunctionChunk>`

**Passe 2** — `compile_stmt` / `compile_expr` :
- Émet le bytecode associé aux spans via `emit(op, span)`
- Les instructions de niveau supérieur dans `main` utilisent `GetVar`/`SetVar` (globaux) ; les instructions imbriquées utilisent des registres

### FunctionChunk

```rust
pub struct FunctionChunk {
    pub bytecode:   Arc<Vec<OpCode>>,
    pub spans:      Arc<Vec<Span>>,    // spans[i] correspond à bytecode[i]
    pub is_fiber:   bool,
    pub max_locals: usize,
}
```

Bytecode et spans sont enveloppés dans `Arc` pour pouvoir être partagés entre les threads de workers HTTP sans copie.

---

## Représentation des Valeurs — NaN-Boxing

Chaque valeur est un seul `Value(u64)` — un mot de 64 bits. XCX utilise le **NaN-boxing** : le motif de bits du NaN silencieux IEEE 754 est réutilisé comme préfixe d'étiquette de type.

```
Disposition des bits : [63..52: exposant/QNAN] [51..48: étiquette de type] [47..0: charge utile]

Float : stocké directement comme bits f64 — N'a PAS le préfixe QNAN_BASE défini
Int   : QNAN_BASE | TAG_INT  | (valeur i48 & 0x0000_FFFF_FFFF_FFFF)
Bool  : QNAN_BASE | TAG_BOOL | (0 ou 1)
Date  : QNAN_BASE | TAG_DATE | (horodatage i48 ms)
Ptr   : QNAN_BASE | TAG_XXX  | (pointeur & 0x0000_FFFF_FFFF_FFFF)
```

### Constantes d'Étiquettes

| Constante | Valeur | Type |
|---|---|---|
| `QNAN_BASE` | `0x7FF0_0000_0000_0000` | marqueur NaN de base |
| `TAG_INT` | `0x0001_0000_0000_0000` | entier signé 48 bits |
| `TAG_BOOL` | `0x0002_0000_0000_0000` | booléen (charge utile 0/1) |
| `TAG_DATE` | `0x0003_0000_0000_0000` | horodatage 48 bits (ms) |
| `TAG_STR` | `0x0004_0000_0000_0000` | pointeur `Arc<Vec<u8>>` |
| `TAG_ARR` | `0x0005_0000_0000_0000` | pointeur `Arc<RwLock<Vec<Value>>>` |
| `TAG_SET` | `0x0006_0000_0000_0000` | pointeur `Arc<RwLock<SetData>>` |
| `TAG_MAP` | `0x0007_0000_0000_0000` | pointeur `Arc<RwLock<Vec<(Value,Value)>>>` |
| `TAG_TBL` | `0x0008_0000_0000_0000` | pointeur `Arc<RwLock<TableData>>` |
| `TAG_FUNC` | `0x0009_0000_0000_0000` | indice de fonction (u32) |
| `TAG_ROW` | `0x000A_0000_0000_0000` | pointeur `Arc<RowRef>` |
| `TAG_JSON` | `0x000B_0000_0000_0000` | pointeur `Arc<RwLock<serde_json::Value>>` |
| `TAG_FIB` | `0x000C_0000_0000_0000` | pointeur `Arc<RwLock<FiberState>>` |
| `TAG_DB` | `0x000D_0000_0000_0000` | pointeur `Arc<DatabaseData>` |

Les charges utiles de pointeurs utilisent seulement les 48 bits inférieurs — valides sur toutes les plateformes x86-64 et AArch64 où les pointeurs de l'espace utilisateur tiennent en 48 bits.

### Comptage de Références

Les valeurs étiquetées par des pointeurs portent des comptages de références `Arc`. La VM les gère manuellement via `inc_ref()` / `dec_ref()` à chaque assignation, retour et modification de collection — assurant que les objets alloués sur le tas sont libérés quand ils ne sont plus référencés, sans ramasse-miettes.

---

## Jeu d'Instructions (OpCodes)

Tous les opcodes sont basés sur des registres : ils réfèrent à des slots de registres `u8` nommés au lieu d'une pile d'opérandes.

### Mouvement de Registres / Variables

| OpCode | Description |
|---|---|
| `LoadConst { dst, idx }` | Charge `constants[idx]` dans le registre `dst` |
| `Move { dst, src }` | Copie le registre `src` vers `dst` |
| `GetVar { dst, idx }` | Charge `globals[idx]` dans `dst` (verrou de lecture sur les globaux) |
| `SetVar { idx, src }` | Stocke `src` dans `globals[idx]` (verrou d'écriture sur les globaux) |

### Arithmétique

Toutes les opérations arithmétiques sont à 3 registres : `dst = src1 OP src2`. Le dispatch de type à l'exécution sélectionne les chemins entier, flottant, concaténation de chaînes, arithmétique de date ou opération d'ensemble.

`Add`, `Sub`, `Mul`, `Div`, `Mod`, `Pow`, `IntConcat` (`++`)

### Comparaisons (résultat Bool)

`Equal`, `NotEqual`, `Greater`, `Less`, `GreaterEqual`, `LessEqual`

### Logique

`And { dst, src1, src2 }`, `Or { dst, src1, src2 }`, `Not { dst, src }`, `Has { dst, src1, src2 }`

### Flux de Contrôle

| OpCode | Description |
|---|---|
| `Jump { target }` | Saut inconditionnel ; incrémente `hot_counts[target]` sur les sauts en arrière |
| `JumpIfFalse { src, target }` | Saute si `src` est `Bool(false)` |
| `JumpIfTrue { src, target }` | Saute si `src` est `Bool(true)` |
| `Call { dst, func_idx, base, arg_count }` | Appelle une fonction ; résultat → `dst` |
| `Return { src }` | Retourne la valeur dans `src` du frame actuel |
| `ReturnVoid` | Retourne sans valeur |
| `Halt` | Arrête l'exécution |

### Collections

| OpCode | Description |
|---|---|
| `ArrayInit { dst, base, count }` | Collecte `count` registres depuis `base` → nouveau tableau dans `dst` |
| `SetInit { dst, base, count }` | Collecte `count` registres → nouvel ensemble dans `dst` |
| `SetRange { dst, start, end, step, has_step }` | Construit un ensemble de plage depuis des valeurs de registres |
| `MapInit { dst, base, count }` | Collecte `count` paires clé-valeur → nouvelle map dans `dst` |
| `TableInit { dst, skeleton_idx, base, row_count }` | Construit une table depuis la constante de schéma de colonnes + valeurs de lignes |

### Opérations d'Ensembles

`SetUnion`, `SetIntersection`, `SetDifference`, `SetSymDifference` — tous à 3 registres, les deux opérandes doivent être `TAG_SET`.

### Dispatch de Méthodes

| OpCode | Description |
|---|---|
| `MethodCall { dst, kind, base, arg_count }` | Dispatch de méthode intégrée via l'enum `MethodKind` — pas de recherche de chaîne à l'exécution |
| `MethodCallCustom { dst, method_name_idx, base, arg_count }` | Dispatch de méthode dynamique (champ JSON, alias) via chaîne de la table des constantes |
| `MethodCallNamed { dst, kind, base, arg_count, names_idx }` | Appel de méthode avec arguments nommés |

`base` pointe vers le registre du receveur ; les arguments sont dans `locals[base+1..base+1+arg_count]`. `MethodKind` est un enum `#[derive(Copy)]` couvrant ~50 méthodes intégrées (`Push`, `Pop`, `Get`, `Insert`, `Update`, `Delete`, `Where`, `Join`, `Sort`, `Format`, `Next`, `IsDone`, `Close`, etc.).

### Opérations sur les Fibres

| OpCode | Description |
|---|---|
| `FiberCreate { dst, func_idx, base, arg_count }` | Alloue `FiberState`, pré-remplit les locaux depuis les args, stocke `Fiber` dans `dst` |
| `Yield { src }` | Suspend la fibre, retourne la valeur dans `src` à l'appelant |
| `YieldVoid` | Suspend une fibre void |

### E/S et Système

| OpCode | Description |
|---|---|
| `Print { src }` | Imprime `locals[src]` vers stdout |
| `Input { dst, ty }` | Lit une ligne depuis stdin → `dst` avec conversion de type |
| `Wait { src }` | Dort pendant `src` millisecondes |
| `HaltAlert { src }` | Imprime un message d'avertissement, continue l'exécution |
| `HaltError { src }` | Imprime une erreur + info span, arrête le frame, incrémente le compteur d'erreurs |
| `HaltFatal { src }` | Imprime une erreur fatale + info span, arrête le frame, incrémente le compteur d'erreurs |
| `TerminalExit` | `std::process::exit(0)` |
| `TerminalClear` | Efface le terminal via une séquence ANSI |
| `TerminalRaw / TerminalNormal` | Active/désactive le mode raw du terminal |
| `TerminalCursor { on }` | Affiche/masque le curseur |
| `TerminalMove { x_src, y_src }` | Déplace le curseur |
| `TerminalWrite { src }` | Écrit sans saut de ligne |
| `InputKey { dst }` | Lit une touche (non-bloquant) |
| `InputKeyWait { dst }` | Lit une touche (bloquant) |
| `InputReady { dst }` | Vérifie si l'entrée est disponible |
| `EnvGet { dst, src }` | Lit une variable d'environnement |
| `EnvArgs { dst }` | Obtient les arguments CLI |

### HTTP

| OpCode | Description |
|---|---|
| `HttpCall { dst, method_idx, url_src, body_src }` | Appel HTTP simple via `ureq`, résultat JSON → `dst` |
| `HttpRequest { dst, arg_src }` | Appel HTTP complet depuis une map de configuration (méthode, url, en-têtes, corps, timeout) → `dst` |
| `HttpRespond { status_src, body_src, headers_src }` | Envoie une réponse HTTP depuis l'intérieur d'une fibre gestionnaire |
| `HttpServe { func_idx, port_src, host_src, workers_src, routes_src }` | Démarre le serveur `tiny_http`, spawne des threads workers |

### Stockage

`StoreWrite`, `StoreRead`, `StoreAppend`, `StoreExists`, `StoreDelete`, `StoreList`, `StoreIsDir`, `StoreSize`, `StoreMkdir`, `StoreGlob`, `StoreZip`, `StoreUnzip`

### JSON

`JsonParse { dst, src }`, `JsonBind { idx, json_src, path_src }`, `JsonBindLocal { dst, json_src, path_src }`, `JsonInject { table_idx, json_src, mapping_src }`, `JsonInjectLocal { table_reg, json_src, mapping_src }`

### Conversions de Types

`CastInt { dst, src }`, `CastFloat { dst, src }`, `CastString { dst, src }`, `CastBool { dst, src }`

### Crypto et Dates

`CryptoHash`, `CryptoVerify`, `CryptoToken`, `DateNow`

### Base de Données

`DatabaseInit { dst, engine_src, path_src, tables_base_reg, table_count }`

---

## Architecture de la VM

**Fichier :** `src/backend/vm.rs`

### État de la VM

```rust
pub struct VM {
    pub globals:     Arc<RwLock<Vec<Value>>>,
    pub error_count: AtomicUsize,
    pub traces:      Arc<RwLock<HashMap<usize, Arc<Trace>>>>,
    pub jit:         Mutex<JIT>,
}
```

`VM` est enveloppée dans `Arc<VM>` et partagée entre les threads de workers HTTP. Chaque worker crée son propre `Executor` avec des locaux privés.

### État de l'Executor

```rust
struct Executor {
    vm:               Arc<VM>,
    ctx:              SharedContext,
    current_spans:    Option<Arc<Vec<Span>>>,
    fiber_yielded:    bool,
    hot_counts:       Vec<usize>,         // compteur de saut en arrière par IP
    recording_trace:  Option<Trace>,      // trace en cours d'enregistrement
    is_recording:     bool,
    trace_cache:      Vec<Option<Arc<Trace>>>, // traces compilées indexées par IP de départ
    http_req:         Option<Arc<Mutex<Option<tiny_http::Request>>>>,
    http_req_val:     Option<Value>,
    terminal_raw_enabled: bool,
}
```

### SharedContext

```rust
pub struct SharedContext {
    pub constants: Arc<Vec<Value>>,
    pub functions: Arc<Vec<FunctionChunk>>,
}
```

`SharedContext` est cloné à faible coût (deux incréments de compteur `Arc`) et passé à chaque thread worker indépendamment. Aucune copie profonde.

### Flux d'Exécution

```
VM::run(main_chunk, ctx)                        [Arc<VM>]
  └─ Executor::run_frame_owned(main_chunk)
       └─ execute_bytecode_inner(bytecode, &mut ip, &mut locals, ...)
            │
            ├─ [Chemin Rapide JIT] si trace_cache[ip].is_some():
            │     execute_trace(trace, ip, locals, globals)
            │     → retourne le prochain IP ou None
            │
            └─ [Chemin Interpréteur] récupère l'opcode, exécute
                 ├─ Continue      → continue normalement
                 ├─ Jump(t)       → ip = t; incrémente hot_counts si en arrière
                 ├─ Return(val)   → quitte le frame, retourne val
                 ├─ Yield(val)    → suspend (fibre), retourne val à l'appelant
                 └─ Halt          → arrête, incrémente error_count
```

---

## Modèle d'Exécution des Fibres

Les fibres sont des **coroutines coopératives**, pas des threads OS.

```rust
pub struct FiberState {
    pub func_id:       usize,
    pub ip:            usize,
    pub locals:        Vec<Value>,    // déplacé lors de la reprise, retourné après
    pub is_done:       bool,
    pub yielded_value: Option<Value>, // cache pour le motif IsDone + Next
}
```

### Séquence de Reprise (`resume_fiber`)

1. Lit `func_id`, `ip` et **déplace** `locals` depuis `FiberState` via `std::mem::take` — pas de clonage
2. Démarre `execute_bytecode` depuis `fiber.ip` avec les locaux déplacés
3. Sur `Yield` : définit `fiber_yielded = true`. Déplace les locaux vers `FiberState`. Met à jour `fiber.ip`. Retourne la valeur produite.
4. Sur `Return` / fin du bytecode : définit `fiber.is_done = true`. Retourne la valeur finale.

La reprise/suspension ne déclenche aucune allocation sur le tas au-delà de la création initiale de `Vec` — seulement des déplacements.

### Motif IsDone / Next

`IsDone` vérifie `FiberState::is_done`, en tenant compte de si `yielded_value` est en cache. `Next` prend la `yielded_value` en cache si présente, ou appelle `resume_fiber`. Cela assure qu'une boucle `for x in fiber` n'avance jamais doublement la fibre.

### Boucle For sur une Fibre (`ForIterType::Fiber`)

Le compilateur émet :
1. `MethodCall(IsDone)` → `JumpIfTrue` pour sortir
2. `MethodCall(Next)` → assigne à la variable de boucle
3. Corps de la boucle
4. `Jump` vers l'étape 1
5. Sur `break` : `MethodCall(Close, base=fiber_reg)` marque la fibre comme terminée avant de sauter

---

## Serveur HTTP (`HttpServe`)

`HttpServe` démarre un serveur `tiny_http::Server` et spawne N threads OS :

```rust
for _ in 0..workers {
    let server  = server.clone();    // Arc<tiny_http::Server>
    let vm      = vm_arc.clone();    // Arc<VM>
    let ctx     = self.ctx.clone();  // SharedContext (deux clones Arc)
    let routes  = routes.clone();    // Arc<Vec<(String, usize)>>
    std::thread::spawn(move || { /* recv → match route → run handler fiber */ });
}
```

Chaque worker exécute son propre `Executor` avec ses propres locaux. Les globaux sont partagés via `Arc<RwLock<Vec<Value>>>`.

### Traitement des Requêtes

Pour chaque requête entrante, le worker :
1. Fait correspondre la clé `"MÉTHODE /chemin"` contre la table des routes (insensible à la casse).
2. Construit un objet JSON `{ method, url, body, ip, headers }` comme `Value::Json`.
3. Stocke la `tiny_http::Request` dans `Arc<Mutex<Option<tiny_http::Request>>>` et la passe à un nouvel `Executor` via `http_req`.
4. Exécute la fibre gestionnaire correspondante de manière synchrone dans cet `Executor` de worker.
5. Quand le gestionnaire appelle `net.respond(...)`, la VM exécute `HttpRespond` qui envoie la réponse et retourne `OpResult::Yield` pour terminer le gestionnaire.
6. Si le gestionnaire sort sans appeler `net.respond`, le worker envoie une réponse de repli `500`.

### Arrêt Gracieux

`SHUTDOWN` est un `pub static AtomicBool` dans `vm.rs`. Un gestionnaire Ctrl+C dans `main.rs` le définit à `true`. Les workers vérifient cela à chaque cycle `recv_timeout(100ms)`. Le thread principal bloque dans une boucle `sleep(500ms)` interrogeant également `SHUTDOWN`. Une fois défini, toutes les boucles sortent et le processus se termine proprement.

---

## Gestion de la Mémoire

- **Pas de ramasse-miettes**. Comptage de références via `Arc` (avec `inc_ref`/`dec_ref` manuel pour les valeurs de pointeurs NaN-boxés).
- **Valeurs scalaires** (Int, Float, Bool, Date, indice de Fonction) : stockées entièrement dans le `u64` — zéro allocation sur le tas.
- **Valeurs de collections** : `Arc<RwLock<T>>` fournit une propriété partagée. Le clonage d'une `Value` de collection incrémente seulement le compteur `Arc`.
- **Mutations** : `.insert()`, `.update()`, `.delete()` acquièrent un verrou d'écriture. Tous les handles vers la même collection voient le changement.
- **Méthodes en lecture seule** : `.size()`, `.get()`, `.contains()` acquièrent un verrou de lecture — plusieurs lecteurs concurrents sont autorisés.

---

## Optimisations de Boucles

Ces opcodes sont émis par le compilateur pour fusionner des motifs de compteurs de boucle courants en instructions uniques, réduisant la surcharge de dispatch et améliorant le traçage JIT :

| OpCode | Description |
|---|---|
| `IncLocal { reg }` | Incrémente l'entier dans le registre `reg` de 1 |
| `IncVar { idx }` | Incrémente le global à `idx` de 1 |
| `LoopNext { reg, limit_reg, target }` | Incrémente `reg`, saute à `target` si `reg <= limit_reg` |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target }` | Incrémente `inc_reg` (compteur séparé, p.ex. indice de tableau), incrémente `reg` (variable de boucle), saut conditionnel |
| `IncVarLoopNext { g_idx, reg, limit_reg, target }` | Comme `IncLocalLoopNext` mais `g_idx` est un compteur global |
| `ArrayLoopNext { idx_reg, size_reg, target }` | Itération combinée d'index de tableau |

### Transformation d'Optimisation

Le compilateur vérifie la dernière instruction émise avant la fin d'un pas de boucle. Si c'est `IncVar` ou `IncLocal`, il les remplace par `IncVarLoopNext` ou `IncLocalLoopNext` respectivement, fusionnant l'incrément avec le test conditionnel de boucle :

```rust
match self.bytecode[len - 1] {
    OpCode::IncVar { idx } => {
        self.bytecode.pop();
        self.emit(OpCode::IncVarLoopNext { g_idx: idx, reg: loop_var_reg, ... });
    }
    OpCode::IncLocal { reg } => {
        self.bytecode.pop();
        self.emit(OpCode::IncLocalLoopNext { inc_reg: reg, ... });
    }
}
```