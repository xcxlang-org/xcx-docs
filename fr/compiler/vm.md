# Machine Virtuelle XCX (VM) — v3.1

La VM XCX est un runtime **basé sur les registres** pour l'exécution du bytecode XCX, augmenté d'un compilateur JIT à traçage construit sur Cranelift.

## Architecture

- **Fichier** : `src/backend/vm.rs`
- **Modèle d'exécution** : boucle Fetch-Decode-Execute (`execute_bytecode`) sur un fichier de registres plat
- **Fichier de registres** : un `Vec<Value>` possédé par frame, indexé par des numéros de slots `u8`
- **Globales** : un unique `Vec<Value>` plat derrière `Arc<RwLock<Vec<Value>>>`, partagé entre tous les threads workers
- **JIT** : `src/backend/jit.rs` — compilateur de code natif basé sur Cranelift pour les traces chaudes

### État de la VM

```rust
pub struct VM {
    pub globals:     Arc<RwLock<Vec<Value>>>,
    pub error_count: AtomicUsize,
    pub traces:      Arc<RwLock<HashMap<usize, Arc<Trace>>>>,
    pub jit:         Mutex<JIT>,
}
```

La `VM` est enveloppée dans `Arc<VM>` et partagée entre les threads workers HTTP. Chaque worker crée son propre `Executor` avec des locaux privés.

### État de l'Exécuteur

```rust
struct Executor {
    vm:               Arc<VM>,
    ctx:              SharedContext,
    current_spans:    Option<Arc<Vec<Span>>>,
    fiber_yielded:    bool,
    hot_counts:       Vec<usize>,         // compteur de sauts arrière par IP
    recording_trace:  Option<Trace>,      // trace en cours d'enregistrement
    is_recording:     bool,
    trace_cache:      Vec<Option<Arc<Trace>>>, // traces compilées indexées par IP de départ
    http_req:         Option<Arc<Mutex<Option<tiny_http::Request>>>>,
    http_req_val:     Option<Value>,
}
```

### SharedContext

```rust
pub struct SharedContext {
    pub constants: Arc<Vec<Value>>,
    pub functions: Arc<Vec<FunctionChunk>>,
}
```

`SharedContext` est cloné à faible coût (deux incréments de pointeur `Arc`) et transmis à chaque thread worker de façon indépendante. Aucune copie profonde n'a lieu.

---

## Représentation des valeurs : NaN-Boxing

Chaque valeur est un unique `Value(u64)` — un mot de 64 bits. XCX utilise le **NaN-boxing** : le motif de bits NaN silencieux IEEE 754 est réutilisé comme préfixe de tag de type.

```
Disposition des bits : [63..52 : exposant/QNAN] [51..48 : tag de type] [47..0 : charge utile]

Float : stocké directement en bits f64 — N'a PAS de préfixe QNAN_BASE
Int   : QNAN_BASE | TAG_INT  | (valeur i48 & 0x0000_FFFF_FFFF_FFFF)
Bool  : QNAN_BASE | TAG_BOOL | (0 ou 1)
Date  : QNAN_BASE | TAG_DATE | (timestamp i48 en ms)
Ptr   : QNAN_BASE | TAG_XXX  | (pointeur & 0x0000_FFFF_FFFF_FFFF)
```

### Constantes de tags

| Constante | Valeur | Type |
|---|---|---|
| `QNAN_BASE` | `0x7FF0_0000_0000_0000` | marqueur NaN de base |
| `TAG_INT`   | `0x0001_0000_0000_0000` | entier signé 48 bits |
| `TAG_BOOL`  | `0x0002_0000_0000_0000` | booléen (charge utile 0/1) |
| `TAG_DATE`  | `0x0003_0000_0000_0000` | timestamp 48 bits (ms) |
| `TAG_STR`   | `0x0004_0000_0000_0000` | pointeur `Arc<String>` |
| `TAG_ARR`   | `0x0005_0000_0000_0000` | pointeur `Arc<RwLock<Vec<Value>>>` |
| `TAG_SET`   | `0x0006_0000_0000_0000` | pointeur `Arc<RwLock<SetData>>` |
| `TAG_MAP`   | `0x0007_0000_0000_0000` | pointeur `Arc<RwLock<Vec<(Value,Value)>>>` |
| `TAG_TBL`   | `0x0008_0000_0000_0000` | pointeur `Arc<RwLock<TableData>>` |
| `TAG_FUNC`  | `0x0009_0000_0000_0000` | index de fonction (u32) |
| `TAG_ROW`   | `0x000A_0000_0000_0000` | pointeur `Arc<RowRef>` |
| `TAG_JSON`  | `0x000B_0000_0000_0000` | pointeur `Arc<RwLock<serde_json::Value>>` |
| `TAG_FIB`   | `0x000C_0000_0000_0000` | pointeur `Arc<RwLock<FiberState>>` |

Les charges utiles des pointeurs n'utilisent que les 48 bits de poids faible — valide sur toutes les plateformes x86-64 et AArch64 où les pointeurs espace utilisateur tiennent en 48 bits.

### Comptage de références pour les valeurs pointeurs

Les valeurs taguées comme pointeurs portent des compteurs de références `Arc`. La VM les gère manuellement via `inc_ref()` / `dec_ref()` à chaque affectation, retour et modification de collection — garantissant que les objets alloués sur le tas (chaînes, tableaux, JSON, fibres, etc.) sont libérés quand ils ne sont plus référencés, sans ramasse-miettes.

---

## Jeu d'instructions (OpCodes)

Tous les opcodes sont basés sur les registres : ils référencent des slots de registre nommés `u8` plutôt qu'une pile d'opérandes.

### Déplacement de registres / variables

| OpCode | Description |
|---|---|
| `LoadConst { dst, idx }` | Charge `constants[idx]` dans le registre `dst` |
| `Move { dst, src }` | Copie le registre `src` dans `dst` |
| `GetVar { dst, idx }` | Charge `globals[idx]` dans `dst` (verrouillage en lecture des globales) |
| `SetVar { idx, src }` | Écrit `src` dans `globals[idx]` (verrouillage en écriture des globales) |

### Arithmétique

Toutes les opérations arithmétiques utilisent 3 registres : `dst = src1 OP src2`. Le dispatch de type à l'exécution sélectionne les chemins entier, flottant, concaténation de chaînes, arithmétique de dates ou opérations ensemblistes.

`Add`, `Sub`, `Mul`, `Div`, `Mod`, `Pow`, `IntConcat` (`++`)

### Comparaison (résultat de type Bool)

`Equal`, `NotEqual`, `Greater`, `Less`, `GreaterEqual`, `LessEqual`

### Logique

`And { dst, src1, src2 }`, `Or { dst, src1, src2 }`, `Not { dst, src }`, `Has { dst, src1, src2 }`

### Contrôle de flux

| OpCode | Description |
|---|---|
| `Jump { target }` | Saut inconditionnel ; incrémente `hot_counts[target]` sur les sauts arrière |
| `JumpIfFalse { src, target }` | Saut si `src` vaut `Bool(false)` |
| `JumpIfTrue { src, target }` | Saut si `src` vaut `Bool(true)` |
| `Call { dst, func_idx, base, arg_count }` | Appel de fonction ; les arguments sont `locals[base..base+arg_count]` ; résultat → `dst` |
| `Return { src }` | Retourne la valeur dans `src` depuis le frame courant |
| `ReturnVoid` | Retour sans valeur |
| `Halt` | Arrêt de l'exécution (explicite ou sur erreur irrécupérable) |

### Collections

| OpCode | Description |
|---|---|
| `ArrayInit { dst, base, count }` | Collecte `count` registres à partir de `base` → nouveau tableau dans `dst` |
| `SetInit { dst, base, count }` | Collecte `count` registres → nouvel ensemble dans `dst` |
| `SetRange { dst, start, end, step, has_step }` | Construit un ensemble rangé à partir de valeurs de registres |
| `MapInit { dst, base, count }` | Collecte `count` paires clé-valeur (registres en paires alternées depuis `base`) → nouvelle map dans `dst` |
| `TableInit { dst, skeleton_idx, base, row_count }` | Construit une table depuis une constante de schéma de colonnes + valeurs de lignes |

### Opérations ensemblistes

`SetUnion`, `SetIntersection`, `SetDifference`, `SetSymDifference` — tous à 3 registres, les deux opérandes doivent être `TAG_SET`.

### Dispatch de méthodes

| OpCode | Description |
|---|---|
| `MethodCall { dst, kind, base, arg_count }` | Dispatch de méthode intégrée par enum `MethodKind` — pas de recherche par chaîne à l'exécution |
| `MethodCallCustom { dst, method_name_idx, base, arg_count }` | Dispatch de méthode dynamique (champ JSON, alias) par chaîne de la table de constantes |

`base` pointe vers le registre récepteur ; les arguments sont `locals[base+1..base+1+arg_count]`. `MethodKind` est un enum `#[derive(Copy)]` couvrant ~50 méthodes intégrées (`Push`, `Pop`, `Get`, `Insert`, `Update`, `Delete`, `Where`, `Join`, `Sort`, `Format`, `Next`, `IsDone`, `Close`, etc.). Le compilateur résout les noms de méthodes en variantes `MethodKind` à la compilation via `map_method_kind()`.

### Opérations sur les fibres

| OpCode | Description |
|---|---|
| `FiberCreate { dst, func_idx, base, arg_count }` | Alloue un `FiberState`, pré-remplit les locaux depuis les arguments, stocke la `Fiber` dans `dst` |
| `Yield { src }` | Suspend la fibre, retourne la valeur dans `src` à l'appelant |
| `YieldVoid` | Suspend une fibre sans valeur |

### E/S et système

| OpCode | Description |
|---|---|
| `Print { src }` | Affiche `locals[src]` sur stdout |
| `Input { dst }` | Lit une ligne depuis stdin → `dst` |
| `Wait { src }` | Attend `src` millisecondes |
| `HaltAlert { src }` | Affiche un message d'alerte, continue l'exécution |
| `HaltError { src }` | Affiche l'erreur + infos de span, arrête le frame, incrémente le compteur d'erreurs |
| `HaltFatal { src }` | Affiche l'erreur fatale + infos de span, arrête le frame, incrémente le compteur d'erreurs |
| `TerminalExit` | `std::process::exit(0)` |
| `TerminalClear` | Efface le terminal via une séquence ANSI ou une commande OS |
| `TerminalRun { dst, cmd_src }` | Exécute une commande externe, résultat → `dst` |
| `EnvGet { dst, src }` | Lit la variable d'environnement nommée par `src` → `dst` |
| `EnvArgs { dst }` | Pousse un `Array<String>` des arguments CLI → `dst` |

### HTTP

| OpCode | Description |
|---|---|
| `HttpCall { dst, method_idx, url_src, body_src }` | Appel HTTP simple (GET/POST/etc.) via `ureq`, résultat JSON → `dst` |
| `HttpRequest { dst, arg_src }` | Appel HTTP complet depuis une map de configuration (méthode, url, en-têtes, corps, timeout) → `dst` |
| `HttpRespond { status_src, body_src, headers_src }` | Envoie une réponse HTTP depuis une fibre gestionnaire ; déclenche un `Yield` pour rendre le contrôle |
| `HttpServe { func_idx, port_src, host_src, workers_src, routes_src }` | Démarre un serveur `tiny_http`, lance les threads workers, bloque le thread principal jusqu'à `SHUTDOWN` |

### Stockage

`StoreWrite { base }`, `StoreRead { dst, base }`, `StoreAppend { base }`, `StoreExists { dst, base }`, `StoreDelete { base }`

### JSON

`JsonParse { dst, src }`, `JsonBind { idx, json_src, path_src }`, `JsonBindLocal { dst, json_src, path_src }`, `JsonInject { table_idx, json_src, mapping_src }`, `JsonInjectLocal { table_reg, json_src, mapping_src }`

### Conversions de types

`CastInt { dst, src }`, `CastFloat { dst, src }`, `CastString { dst, src }`, `CastBool { dst, src }`

### Cryptographie et dates

`CryptoHash { dst, pass_src, alg_src }`, `CryptoVerify { dst, pass_src, hash_src, alg_src }`, `CryptoToken { dst, len_src }`, `DateNow { dst }`

### Optimisations de boucles

Ces opcodes sont émis par le compilateur pour fusionner les motifs courants de compteurs de boucle en instructions uniques, réduisant le coût de dispatch et améliorant la traçabilité JIT.

| OpCode | Description |
|---|---|
| `IncLocal { reg }` | Incrémente l'entier dans le registre `reg` de 1 |
| `IncVar { idx }` | Incrémente la globale à l'index `idx` de 1 |
| `LoopNext { reg, limit_reg, target }` | Incrémente `reg`, saute vers `target` si `reg <= limit_reg`, sinon continue |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target }` | Incrémente `inc_reg` (compteur séparé, ex. index de tableau), incrémente `reg` (variable de boucle), saut conditionnel |
| `IncVarLoopNext { g_idx, reg, limit_reg, target }` | Comme `IncLocalLoopNext` mais `g_idx` est un compteur global |

---

## Compilateur JIT à traçage

### Vue d'ensemble

XCX 2.2 inclut un **JIT à traçage** qui compile automatiquement les boucles chaudes en code machine natif grâce au framework de génération de code [Cranelift](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift).

Le JIT est entièrement transparent pour le programmeur — il s'active automatiquement et revient à l'interpréteur en cas de garde de type ou d'opération non supportée.

### Détection des traces

À chaque `Jump { target }` où `target < current_ip` (saut arrière, c'est-à-dire un retour de boucle) :

1. `hot_counts[target]` est incrémenté.
2. Quand `hot_counts[target] >= 50`, une nouvelle `Trace` est démarrée avec `start_ip = target`.
3. L'enregistrement de la trace est conditionné : il ne commence que si `trace_cache[target].is_none()` (aucune trace compilée n'existe encore) et si `is_recording` est faux.

### Enregistrement des traces

Tant que `is_recording` est vrai, l'interpréteur exécute chaque opcode normalement **et** enregistre un `TraceOp` spécialisé pour les types à l'exécution courants. Par exemple :

- Un `Add` sur deux registres entiers enregistre `GuardInt { reg: src1 }`, `GuardInt { reg: src2 }`, `AddInt { dst, src1, src2 }` — pas un `Add` générique.
- Un `JumpIfFalse` **non** pris enregistre `GuardTrue { reg: src, fail_ip: target }` — affirmant que la branche est toujours fausse.
- Un `JumpIfFalse` **pris** enregistre `GuardFalse { reg: src, fail_ip: next_ip }`.

Si un opcode ne peut pas être tracé (appels de méthodes complexes, opérations sur chaînes, etc.), l'enregistrement est abandonné et `is_recording` repasse à faux.

### Compilation des traces

Quand la boucle tracée s'exécute à nouveau jusqu'à `start_ip`, la `Trace` complète est transmise à la fonction Cranelift `JIT::compile()` (`src/backend/jit.rs`). Cranelift compile la séquence de `TraceOp` en une fonction native avec la signature :

```rust
unsafe extern "C" fn(
    locals_ptr: *mut Value,
    globals_ptr: *mut Value,
    consts_ptr:  *const Value,
) -> i32
```

La valeur de retour est la prochaine IP à reprendre (0 = continuer normalement, positif = IP de sortie latérale, négatif = arrêt).

Le pointeur de fonction compilée est stocké dans `Trace::native_ptr` (un `AtomicPtr<u8>`) et l'`Arc<Trace>` est inséré à la fois dans `vm.traces` (partagé globalement) et dans `trace_cache` (chemin rapide par exécuteur).

### Exécution des traces

À chaque itération de la boucle de dispatch, avant de récupérer le prochain opcode :

```
if trace_cache[current_ip].is_some() {
    execute_trace(trace, ip, locals, &mut glbs)
    continue
}
```

Si une fonction native compilée est disponible (`native_ptr != null`), elle est appelée via `transmute` directement — court-circuitant entièrement l'interpréteur pour tout le corps de la boucle. Si le JIT n'a pas encore compilé, le chemin `TraceOp` interprété est utilisé comme étape intermédiaire.

### Variantes de TraceOp

| Variante | Description |
|---|---|
| `LoadConst`, `Move` | Déplacements de registres avec valeurs constantes |
| `AddInt/SubInt/MulInt/DivInt/ModInt` | Arithmétique entière avec IP d'échec pour division/modulo par zéro |
| `AddFloat/SubFloat/MulFloat/DivFloat/ModFloat` | Arithmétique flottante |
| `CmpInt / CmpFloat` | Comparaison avec un code de condition `cc: u8` |
| `GuardInt / GuardFloat` | Garde de type — quitte la trace si le registre est du mauvais type |
| `GuardTrue / GuardFalse` | Garde de branche — quitte la trace sur direction de branche inattendue |
| `CastIntToFloat` | Élargissement d'un registre entier en flottant |
| `IncLocal / IncVar` | Incrément d'un registre unique / d'une globale |
| `LoopNextInt` | Incrément combiné + saut conditionnel pour les boucles rangées |
| `IncVarLoopNext / IncLocalLoopNext` | Variantes fusionnées pour les boucles tableau et for-range |
| `GetVar / SetVar` | Accès aux variables globales |
| `And / Or / Not` | Logique booléenne |
| `Jump` | Saut inconditionnel (déclenche la détection de retour de boucle dans la trace) |

---

## Flux d'exécution

```
VM::run(main_chunk, ctx)                        [Arc<VM>]
  └─ Executor::run_frame_owned(main_chunk)
       └─ execute_bytecode(bytecode, &mut ip, &mut locals)
            │
            ├─ [Chemin rapide JIT] si trace_cache[ip].is_some() :
            │     execute_trace(trace, ip, locals, globals)
            │     → retourne la prochaine IP ou None
            │
            └─ [Chemin interpréteur] récupère l'opcode, l'exécute
                 ├─ Continue      → avance ip normalement
                 ├─ Jump(t)       → ip = t ; incrémente hot_counts si saut arrière
                 ├─ Return(val)   → quitte le frame, retourne val
                 ├─ Yield(val)    → suspend (fibre), retourne val à l'appelant
                 └─ Halt          → arrêt, incrémente error_count
```

Les fonctions sont appelées via `run_frame(func_id, params)`, qui crée un vecteur `locals` pré-dimensionné à `chunk.max_locals`. `current_spans` est échangé avec la table de spans de la fonction appelée et restauré au retour.

---

## Modèle d'exécution des fibres

Les fibres sont des **coroutines coopératives**, pas des threads.

```rust
pub struct FiberState {
    pub func_id:       usize,
    pub ip:            usize,
    pub locals:        Vec<Value>,    // déplacé pendant la reprise, replacé ensuite
    pub is_done:       bool,
    pub yielded_value: Option<Value>, // valeur mise en cache pour le motif IsDone + Next
}
```

### Séquence de reprise (`resume_fiber`)

1. Lire `func_id`, `ip` et **déplacer** `locals` hors de `FiberState` via `std::mem::take` — pas de clone.
2. Exécuter `execute_bytecode` depuis `fiber.ip` avec les locaux déplacés.
3. Sur `Yield` : mettre `fiber_yielded = true`. Replacer les locaux dans `FiberState`. Mettre à jour `fiber.ip`. Retourner la valeur cédée.
4. Sur `Return` / fin du bytecode : mettre `fiber.is_done = true`. Retourner la valeur finale.

La reprise/suspension n'implique aucune allocation sur le tas au-delà de la création initiale du `Vec` — uniquement des déplacements.

### Motif IsDone / Next

`IsDone` vérifie `FiberState::is_done`, en tenant compte de la présence éventuelle d'une `yielded_value` en cache. `Next` prend la `yielded_value` en cache si elle est présente (d'une reprise précédente déjà exécutée), ou appelle `resume_fiber`. Cela garantit qu'une boucle `for x in fiber` n'avance jamais la fibre deux fois.

### Boucle for sur une fibre (`ForIterType::Fiber`)

Le compilateur émet :
1. `MethodCall(IsDone)` → `JumpIfTrue` vers la sortie
2. `MethodCall(Next)` → affectation à la variable de boucle
3. Corps de la boucle
4. `Jump` vers l'étape 1
5. Sur `break` : `MethodCall(Close, base=fiber_reg)` marque la fibre comme terminée avant de sauter

---

## Serveur HTTP (`HttpServe`)

`HttpServe` démarre un `tiny_http::Server` et lance N threads OS :

```rust
for _ in 0..workers {
    let server  = server.clone();    // Arc<tiny_http::Server>
    let vm      = vm_arc.clone();    // Arc<VM>
    let ctx     = self.ctx.clone();  // SharedContext (deux clones Arc)
    let routes  = routes.clone();    // Arc<Vec<(String, usize)>>
    std::thread::spawn(move || { /* recv → match route → run handler fiber */ });
}
```

Chaque worker exécute son propre `Executor` avec ses propres locaux. Les globales sont partagées via `Arc<RwLock<Vec<Value>>>`.

### Gestion des requêtes

Pour chaque requête entrante, le worker :
1. Fait correspondre la clé `"METHODE /chemin"` à la table de routes (insensible à la casse).
2. Construit un objet JSON `{ method, url, body, ip, headers }` sous forme de `Value::Json`.
3. Stocke la `tiny_http::Request` dans `Arc<Mutex<Option<tiny_http::Request>>>` et la transmet à un `Executor` frais via `http_req`.
4. Exécute la fibre gestionnaire correspondante de façon synchrone dans l'`Executor` du worker.
5. Quand le gestionnaire appelle `net.respond(...)`, la VM exécute `HttpRespond` qui envoie la réponse et retourne `OpResult::Yield` pour terminer le gestionnaire.
6. Si le gestionnaire se termine sans appeler `net.respond`, le worker envoie une réponse de repli `500`.

### Arrêt gracieux

`SHUTDOWN` est un `pub static AtomicBool` dans `vm.rs`. Un gestionnaire Ctrl+C dans `main.rs` le passe à `true`. Les workers le vérifient à chaque cycle `recv_timeout(100ms)`. Le thread principal se bloque dans une boucle `sleep(500ms)` qui scrute également `SHUTDOWN`. Une fois défini, toutes les boucles se terminent et le processus s'arrête proprement.

---

## Compilateur (`src/backend/mod.rs`)

### Allocation de registres

`FunctionCompiler` suit le prochain registre disponible avec `next_local: usize` :

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

Les locaux (variables nommées) se voient attribuer un slot via `define_local(id, slot)` et sont stockés dans `scopes: Vec<HashMap<StringId, usize>>`. Les temporaires utilisent `push_reg()`/`pop_reg()` — ils sont réutilisés dès que le résultat de l'expression est consommé.

`max_locals_used` est enregistré afin que `FunctionChunk::max_locals` puisse pré-allouer un vecteur de locaux de taille exacte lors de l'appel de la fonction.

### Déduplication des constantes

`CompileContext::add_constant` déduplique les constantes chaînes via `string_constants: HashMap<String, usize>`. Les constantes chaînes en double (ex. `"insert"` apparaissant plusieurs fois comme argument de nom de méthode) réutilisent le même slot de la table de constantes.

### Compilation en deux passes

**Passe 1** — `register_globals_recursive` :
- Attribue un indice de slot à chaque variable globale et instance de déclaration de fibre
- Attribue un indice de fonction à chaque fonction/fibre
- Pré-alloue des slots `FunctionChunk` vides dans `functions: Vec<FunctionChunk>`

**Passe 2** — `compile_stmt` / `compile_expr` :
- Émet le bytecode couplé aux spans via `emit(op, span)`
- Les instructions de niveau supérieur dans `main` utilisent `GetVar`/`SetVar` (globales) ; les instructions imbriquées utilisent des registres

### `FunctionChunk`

```rust
pub struct FunctionChunk {
    pub bytecode:   Arc<Vec<OpCode>>,
    pub spans:      Arc<Vec<Span>>,    // spans[i] correspond à bytecode[i]
    pub is_fiber:   bool,
    pub max_locals: usize,
}
```

Le bytecode et les spans sont enveloppés dans `Arc` pour pouvoir être partagés entre les threads workers HTTP sans copie.

---

## Rapport d'erreurs à l'exécution

Chaque erreur d'exécution dans la VM ajoute `self.current_span_info(ip)` qui retourne `" [line: X, col: Y]"` en consultant `current_spans[ip - 1]`. Exemple :

```
ERROR: R303: Array index out of bounds: 5 [line: 14, col: 7]
```

`current_spans` est échangé avec le bon `Arc<Vec<Span>>` à chaque appel de `run_frame` et restauré à la sortie.

---

## Modèle mémoire

- **Pas de ramasse-miettes**. Comptage de références via `Arc` (avec `inc_ref`/`dec_ref` manuels pour les valeurs pointeurs NaN-boxées).
- **Valeurs scalaires** (Int, Float, Bool, Date, index de fonction) : stockées entièrement dans le `u64` — aucune allocation sur le tas.
- **Valeurs de collections** : `Arc<RwLock<T>>` fournit la propriété partagée. Cloner une `Value` de collection n'incrémente que le compteur `Arc`.
- **Mutations** : `.insert()`, `.update()`, `.delete()` acquièrent un verrou en écriture. Tous les handles sur la même collection voient le changement.
- **Méthodes en lecture seule** : `.size()`, `.get()`, `.contains()` acquièrent un verrou en lecture — plusieurs lecteurs concurrents sont autorisés.

---

## Contrôles de sécurité

### Protection SSRF réseau (`is_safe_url`)

Cibles bloquées :
- URLs `file://`
- `169.254.x.x` (lien-local / endpoint de métadonnées AWS)
- Plages privées : `10.x`, `192.168.x`, `172.16–31.x` (sauf localhost)

Appliqué à `HttpCall` et `HttpRequest`.

### Limite de taille du corps HTTP

Dans `HttpServe`, le corps de la réponse est vérifié après `into_string()`. S'il dépasse **10 Mo**, une erreur JSON `413` est retournée à la place du corps réel.

### En-têtes CORS

Toutes les réponses de `HttpServe` incluent automatiquement :
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

Les requêtes de prévol `OPTIONS` reçoivent une réponse `204` sans invoquer aucun gestionnaire.