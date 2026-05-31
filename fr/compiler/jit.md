# JIT par Traçage XCX (Cranelift) — Documentation

> **Fichier :** `src/backend/jit.rs`  
> XCX 3.1 inclut un **JIT en double mode** qui compile automatiquement les boucles chaudes et les fonctions fréquemment appelées en code machine natif en utilisant le framework [Cranelift](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift).

---

## Table des Matières

1. [Vue d'Ensemble](#vue-densemble)
2. [Deux Modes JIT](#deux-modes-jit)
3. [JIT par Traçage — Détection des Boucles Chaudes](#jit-par-traçage--détection-des-boucles-chaudes)
4. [Enregistrement de Traces](#enregistrement-de-traces)
5. [Compilation des Traces](#compilation-des-traces)
6. [Exécution des Traces](#exécution-des-traces)
7. [JIT de Méthode — Compilation de Fonctions](#jit-de-méthode--compilation-de-fonctions)
8. [Variantes de TraceOp](#variantes-de-traceop)
9. [Fonctions C Exportées](#fonctions-c-exportées)
10. [Configuration JIT](#configuration-jit)

---

## Vue d'Ensemble

Le JIT est **complètement transparent** pour le développeur — il s'active automatiquement et se replie sur l'interpréteur en cas de gardes de type ou d'opérations non supportées.

Il y a deux chemins de compilation indépendants :

```
[JIT par Traçage] — pour les boucles chaudes
  L'interpréteur exécute la boucle
      ↓ (50 itérations)
  L'enregistrement de traces commence
      ↓ (la boucle retourne à l'IP de départ)
  La trace est compilée par Cranelift → fonction C native (signature à 3 args)
      ↓ (prochaine itération)
  Code natif exécuté directement (contournement de l'interpréteur)
      ↓ (garde échouée)
  Repli vers l'interpréteur à l'IP correct

[JIT de Méthode] — pour les fonctions fréquemment appelées sans boucles
  La fonction est appelée
      ↓ (10 appels)
  Bytecode complet compilé par Cranelift → fonction C native (signature à 5 args)
      ↓ (prochain appel)
  Code natif appelé directement (contourne l'interpréteur + la machinerie de traçage)
```

---

## Deux Modes JIT

### JITFunction (Traçage)

```rust
pub type JITFunction = unsafe extern "C" fn(
    *mut VMValue,   // locals_ptr
    *mut VMValue,   // globals_ptr
    *const VMValue, // consts_ptr
) -> i32            // next_ip (0 = continuer, >0 = IP de sortie latérale, <0 = arrêt)
```

Utilisé pour les traces de boucles chaudes. La valeur de retour est le prochain pointeur d'instruction.

### MethodJitFunction (Méthode)

```rust
pub type MethodJitFunction = unsafe extern "C" fn(
    *mut VMValue,   // locals_ptr
    *mut VMValue,   // globals_ptr
    *const VMValue, // consts_ptr
    *mut VM,        // vm_ptr
    *mut Executor,  // executor_ptr
) -> u64            // valeur de retour comme bits Value bruts
```

Utilisé pour les fonctions complètes compilées. Retourne la valeur de retour de la fonction directement comme bits NaN-boxés. Nécessite l'accès à la VM et à l'Executor pour les appels récursifs et le dispatch de méthodes.

---

## JIT par Traçage — Détection des Boucles Chaudes

Sur chaque `Jump { target }` où `target < ip_actuel` (saut en arrière, c'est-à-dire arête de retour de boucle) :

1. `hot_counts[target]` est incrémenté
2. Quand `hot_counts[target] >= 50`, une nouvelle `Trace` est démarrée avec `start_ip = target`
3. L'enregistrement de traces est gardé : ne commence que si `trace_cache[target].is_none()` (aucune trace compilée) et que `is_recording` est faux

```rust
fn check_start_recording(&mut self, target_ip: usize, threshold: usize) {
    if target_ip < self.hot_counts.len() {
        let hc = unsafe { self.hot_counts.get_unchecked_mut(target_ip) };
        *hc += 1;
        if *hc >= threshold && !self.is_recording && self.trace_cache[target_ip].is_none() {
            self.recording_trace = Some(Trace { ops: vec![], start_ip: target_ip, ... });
            self.is_recording = true;
        }
    }
}
```

Le seuil est de **50** pour l'opcode `Jump` principal et de **1000** pour les opcodes de boucle fusionnés (`LoopNext`, `IncVarLoopNext`, `IncLocalLoopNext`).

---

## Enregistrement de Traces

Quand `is_recording` est vrai, l'interpréteur exécute chaque opcode normalement **et** enregistre un `TraceOp` spécialisé pour les types d'exécution actuels. Exemples :

- `Add` sur deux registres entiers enregistre `GuardInt { reg: src1 }`, `GuardInt { reg: src2 }`, `AddInt { dst, src1, src2 }` — pas un `Add` générique
- `JumpIfFalse` qui n'est **pas** pris enregistre `GuardTrue { reg: src, fail_ip: target }` — affirmant que la branche est toujours fausse
- `JumpIfFalse` qui **est** pris enregistre `GuardFalse { reg: src, fail_ip: next_ip }`

Si un opcode ne peut pas être tracé (appels de méthodes complexes, opérations sur les chaînes, etc.), l'enregistrement est abandonné et `is_recording` est remis à faux.

### Qu'est-ce qui est Traçable ?

| Traçable | Non Traçable |
|---|---|
| Arithmétique Int/Float | Méthodes d'objets (sauf ArraySize, ArrayGet, ArrayPush, ArrayUpdate, SetSize, SetContains) |
| Comparaisons Int/Float | Opérations sur les chaînes |
| Logique booléenne | Appels de fonctions |
| Accès/modification de globaux | E/S, HTTP |
| Incréments de compteurs | Opérations de fibres |
| Opérations de tableaux : size, get, push, update | |
| Opérations d'ensembles : size, contains | |
| RandomInt, RandomFloat, RandomChoice | |
| Pow, IntConcat, Has | |
| CastIntToFloat | |

---

## Compilation des Traces

Quand la trace retourne à `start_ip`, la `Trace` complète est passée à `JIT::compile()` dans Cranelift. Cranelift compile la séquence de `TraceOp` en une fonction native avec la signature `JITFunction` (3 paramètres, retour `i32`).

Le pointeur de fonction compilée est stocké dans `Trace::native_ptr` (`AtomicPtr<u8>`) et l'`Arc<Trace>` est inséré à la fois dans `vm.traces` (partagé globalement) et `trace_cache` (chemin rapide par executor).

### Paramètres du Compilateur Cranelift

```rust
flag_builder.set("opt_level", "speed").unwrap();
flag_builder.set("use_colocated_libcalls", "false").unwrap();
flag_builder.set("is_pic", "false").unwrap();
flag_builder.set("regalloc_checker", "false").unwrap();
```

---

## Exécution des Traces

À chaque itération de la boucle de dispatch, avant de récupérer le prochain opcode :

```rust
if !self.is_recording && current_ip < self.trace_cache.len() {
    if let Some(trace) = &self.trace_cache[current_ip] {
        let jit_res = self.execute_trace(trace, ip, locals, glbs.as_mut().unwrap());
        if let Some(res) = jit_res { return res; }
        continue;
    }
}
```

Si une fonction native compilée est disponible (`native_ptr != null`), elle est appelée directement via `transmute` — contournant complètement l'interpréteur pour tout le corps de la boucle. Si le JIT n'a pas encore compilé, le chemin `TraceOp` interprété est utilisé comme étape intermédiaire.

---

## JIT de Méthode — Compilation de Fonctions

Les fonctions **sans boucles** (`has_loops = false`) et avec moins de 500 instructions bytecode sont éligibles à la compilation JIT de méthode.

### Déclencheur

Après **10 appels** à la même fonction, la compilation JIT est déclenchée de manière asynchrone :

```rust
let count = chunk.call_count.fetch_add(1, Ordering::Relaxed);
if count == 10 {
    let mut jit = vm_copy.jit.lock();
    match jit.compile_method(func_id_copy, &chunk_copy, &self.ctx.constants) {
        Ok(ptr) => {
            chunk_copy.jit_ptr.store(ptr as *mut u8, Ordering::Release);
        }
        Err(_) => {}
    }
}
```

Le pointeur compilé est stocké dans `FunctionChunk::jit_ptr` (`Arc<AtomicPtr<u8>>`), partagé entre tous les threads.

### Exécution du Chemin Rapide

À chaque appel `run_frame_with_guard`, le pointeur JIT est vérifié en premier :

```rust
let jit_ptr = chunk.jit_ptr.load(Ordering::Relaxed);
if !jit_ptr.is_null() && !self.is_recording {
    let jit_fn: MethodJitFunction = unsafe { std::mem::transmute(jit_ptr) };
    // Prépare les locaux depuis les params, appelle la fonction native directement
    let res_bits = unsafe { jit_fn(locals.as_mut_ptr(), glbs_ptr, consts.as_ptr(), vm_ptr, executor_ptr) };
    return Some(Value(res_bits));
}
```

Cela contourne toute la boucle d'interpréteur, la machinerie de traçage, et l'allocation `hot_counts`/`trace_cache` pour les fonctions compilées.

### `compile_method` — Compilation Complète du Bytecode

`JIT::compile_method` compile un `FunctionChunk` complet en code natif. Il gère un plus grand sous-ensemble d'opcodes que la compilation de traces :

- Tous les opcodes arithmétiques, de comparaison et logiques (entiers uniquement)
- `LoadConst`, `Move`, `GetVar`, `SetVar` avec comptage de références complet
- Flux de contrôle : `Jump`, `JumpIfFalse`, `JumpIfTrue`, `Return`, `ReturnVoid`
- Opcodes de boucle : `LoopNext`, `IncLocalLoopNext`, `IncVarLoopNext`, `ArrayLoopNext`
- `MethodCall` — dispatché via la fonction extern C `xcx_jit_method_dispatch`
- `Call` — appels récursifs via `xcx_jit_call_recursive` (seuls les appels auto-récursifs sont supportés inline ; les autres appels de fonctions reviennent au chemin interpréteur)

Les opcodes non supportés provoquent un `return` précoce avec une valeur zéro (faux), repliant gracieusement.

### Comptage de Références dans le JIT de Méthode

Contrairement au JIT par Traçage (qui saute le comptage de références pour la vitesse), le JIT de Méthode inclut des appels complets `inc_ref`/`dec_ref` via les fonctions extern C `xcx_jit_inc_ref` et `xcx_jit_dec_ref`. C'est nécessaire car les fonctions compilées peuvent détenir et relâcher des valeurs allouées sur le tas pendant leur durée de vie.

Les vérifications de pointeurs sont faites inline en inspectant les bits d'étiquette avant d'appeler les helpers de comptage de références, évitant la surcharge d'appels de fonctions inutiles pour les valeurs scalaires.

---

## Mise en Pool du Chemin Rapide

L'executor maintient trois pools pour éviter les allocations O(N) dans les fonctions profondément récursives ou fréquemment appelées :

```rust
hot_counts_pool:   Vec<Vec<usize>>,
trace_cache_pool:  Vec<Vec<Option<Arc<Trace>>>>,
locals_pool:       Vec<Vec<Value>>,
```

Lors de l'entrée dans `run_frame_with_guard`, les vecteurs sont récupérés depuis le pool (ou fraîchement alloués). Au retour, ils sont retournés au pool. Cela élimine la plupart des allocations sur le tas par appel dans les chemins de code chauds.

Le `locals_pool` est également utilisé par `xcx_jit_call_recursive` pour le chemin rapide des fonctions compilées auto-récursives.

---

## Variantes de TraceOp

### Mouvement de Données

| Variante | Description |
|---|-----|
| `LoadConst { dst, val }` | Charge une constante dans un registre |
| `Move { dst, src }` | Copie un registre |
| `GetVar { dst, idx }` | Charge un global |
| `SetVar { idx, src }` | Stocke un global |

### Arithmétique Entière

| Variante | Description |
|---|---|
| `AddInt / SubInt / MulInt` | Opérations de base (avec enroulement) |
| `DivInt / ModInt` | Avec `fail_ip` pour division par zéro / débordement |
| `PowInt` | Exponentiation via fonction extern C |
| `IntConcat` | Concaténation de chiffres (123 ++ 456 = 123456) |

### Arithmétique Flottante

| Variante | Description |
|---|---|
| `AddFloat / SubFloat / MulFloat` | Opérations de base |
| `DivFloat / ModFloat` | Avec `fail_ip` pour division par zéro |
| `PowFloat` | Exponentiation via fonction extern C |
| `CastIntToFloat` | Conversion int → float |

### Gardes de Type

| Variante | Description |
|---|---|
| `GuardInt { reg, ip }` | Sortie latérale vers `ip` si le registre n'est pas Int |
| `GuardFloat { reg, ip }` | Sortie latérale vers `ip` si le registre n'est pas Float |
| `GuardTrue { reg, fail_ip }` | Sortie latérale si la valeur booléenne est fausse |
| `GuardFalse { reg, fail_ip }` | Sortie latérale si la valeur booléenne est vraie |

### Comparaisons

| Variante | Description |
|---|---|
| `CmpInt { dst, src1, src2, cc }` | Comparaison d'entiers avec code de condition |
| `CmpFloat { dst, src1, src2, cc }` | Comparaison de flottants avec code de condition |

**Codes de Condition (cc) :**

| cc | IntCC | FloatCC |
|---|---|---|
| 0 | Equal | Equal |
| 1 | NotEqual | NotEqual |
| 2 | SignedGreaterThan | GreaterThan |
| 3 | SignedLessThan | LessThan |
| 4 | SignedGreaterThanOrEqual | GreaterThanOrEqual |
| 5 | SignedLessThanOrEqual | LessThanOrEqual |

### Contrôle de Boucle

| Variante | Description |
|---|---|
| `LoopNextInt { reg, limit_reg, target, exit_ip }` | Incrément + saut conditionnel pour boucles de plage |
| `IncVarLoopNext { g_idx, reg, limit_reg, target, exit_ip }` | Compteurs globaux + boucle de plage |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target, exit_ip }` | Compteurs locaux + boucle de plage |
| `IncLocal { reg }` | Incrément simple de variable locale |
| `IncVar { g_idx }` | Incrément simple de variable globale |
| `Jump { target_ip }` | Saut inconditionnel (déclenche la détection de retour de boucle) |

### Collections

| Variante | Description |
|---|---|
| `ArraySize { dst, src }` | Obtient la taille du tableau |
| `ArrayGet { dst, arr_reg, idx_reg, fail_ip }` | Obtient un élément (avec vérification des bornes) |
| `ArrayPush { arr_reg, val_reg }` | Ajoute un élément au tableau |
| `ArrayUpdate { arr_reg, idx_reg, val_reg, fail_ip }` | Met à jour un élément à l'index (avec vérification des bornes) |
| `SetSize { dst, src }` | Obtient la taille de l'ensemble |
| `SetContains { dst, set_reg, val_reg }` | Vérifie l'appartenance à l'ensemble |

### Aléatoire

| Variante | Description |
|---|---|
| `RandomInt { dst, min, max, step, has_step }` | Entier aléatoire avec plage/pas |
| `RandomFloat { dst, min, max, step, has_step, step_is_float }` | Flottant aléatoire avec plage/pas |
| `RandomChoice { dst, src }` | Élément aléatoire depuis une collection |

### Logique

`And / Or / Not` — opérations sur les bits booléens

---

## Fonctions C Exportées

Le JIT appelle des fonctions Rust externes via C ABI pour les opérations qui ne peuvent pas être trivially inlinées. Toutes sont enregistrées dans `JITBuilder` pendant l'initialisation du JIT.

### Arithmétique et Mathématiques

```rust
#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_random_int(min: i64, max: i64, step: i64, has_step: bool) -> i64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_random_float(min: f64, max: f64, step: f64, has_step: bool) -> f64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_pow_int(a: i64, b: i64) -> i64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_pow_float(a: f64, b: f64) -> f64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_int_concat(a: i64, b: i64) -> i64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_has(container: Value, item: Value) -> bool

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_random_choice(col: Value) -> Value
```

### Opérations sur les Collections

```rust
#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_array_size(arr: Value) -> i64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_array_get(arr: Value, idx: i64) -> Value

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_array_push(arr: Value, val: Value)

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_array_update(arr: Value, idx: i64, val: Value) -> i32
// Retourne 1 en cas de succès, 0 hors limites (déclenche une sortie latérale JIT)

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_set_size(set: Value) -> i64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_set_contains(set: Value, val: Value) -> bool
```

### Comptage de Références

```rust
#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_inc_ref(v: Value)
// Incrémente le comptage de références Arc si la valeur est un type pointeur

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_dec_ref(v: Value)
// Décrémente le comptage de références Arc si la valeur est un type pointeur ; peut libérer de la mémoire
```

Utilisé exclusivement par le JIT de Méthode pour gérer les durées de vie des valeurs allouées sur le tas. Le JIT par Traçage saute le comptage de références pour des performances maximales de boucle.

### Dispatch de Méthodes et Appels Récursifs

```rust
#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_method_dispatch(
    dst: u8,
    kind: u8,           // MethodKind casté en u8
    receiver: Value,
    args_ptr: *const Value,
    arg_count: u8,
    locals_ptr: *mut Value,
    executor_ptr: *mut Executor,
)
// Dispatche un appel de méthode intégrée depuis l'intérieur d'une fonction compilée.
// Délègue à Executor::handle_method_call ou handle_database_method.

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_call_recursive(
    func_id_idx: usize,
    params_ptr: *const Value,
    params_count: u8,
    vm_ptr: *const VM,
    executor_ptr: *mut Executor,
    globals_ptr: *mut Value,
) -> u64
// Appelle une fonction depuis l'intérieur d'une fonction compilée.
// Chemin rapide : si la fonction cible est aussi compilée JIT, l'appelle directement
//                en utilisant les locaux du pool de l'executor (évite le verrouillage).
// Chemin lent : revient à run_frame_with_guard pour les fonctions non compilées.
```

---

## Configuration JIT

```rust
pub struct JIT {
    builder_context: FunctionBuilderContext,
    pub ctx:         codegen::Context,
    module:          JITModule,
}
```

Initialisation :
1. Crée `JITBuilder` avec ISA natif (auto-détection de l'hôte via `cranelift_native`)
2. Enregistre tous les symboles C externes (arithmétique, collections, comptage de références, dispatch)
3. Définit `opt_level: "speed"` pour des performances maximales
4. Crée `JITModule` gérant la mémoire exécutable

### Compilation de traces (`compile`)

1. Efface le contexte (`module.clear_context`)
2. Déclare une fonction avec la signature `JITFunction` : `(locals, globals, consts) -> i32`
3. Détecte la présence de boucle (`has_loop`) — si un `LoopNextInt`, `IncVarLoopNext`, etc. est présent, enveloppe les ops dans un bloc de boucle Cranelift
4. Construit l'IR Cranelift pour chaque `TraceOp`
5. Compile et finalise les définitions
6. Retourne le pointeur de code (`get_finalized_function`)

### Compilation de méthode (`compile_method`)

1. Efface le contexte (`module.clear_context`)
2. Déclare une fonction avec la signature `MethodJitFunction` : `(locals, globals, consts, vm, executor) -> u64`
3. Pré-analyse le bytecode pour tous les cibles de saut pour créer le bon nombre de blocs Cranelift
4. Construit l'IR Cranelift pour chaque `OpCode` supporté
5. Scelle tous les blocs et finalise
6. Retourne le pointeur de code ; stocké dans `FunctionChunk::jit_ptr`

---

## Macros NaN-boxing dans le JIT

Le JIT utilise des macros pour travailler avec des valeurs NaN-boxées dans l'IR Cranelift :

```rust
// Décompresse un entier (extension de signe depuis 48 bits)
macro_rules! unpack_int {
    ($val:expr) => {{
        let shl = b.ins().ishl_imm($val, 16);
        b.ins().sshr_imm(shl, 16)
    }};
}

// Compresse un entier
macro_rules! pack_int {
    ($raw:expr) => {{
        let lo = b.ins().band($raw, mask_48);
        b.ins().bor(qnan_tag_int, lo)
    }};
}

// Optimisation : Pour un Int NaN-boxé, l'incrément est un simple iadd_imm(val, 1)
// sur le mot entier de 64 bits — les bits d'étiquette restent inaffectés !
let lnxt_bits = b.ins().iadd_imm(lv, 1);
b.ins().store(trusted(), lnxt_bits, la, 0);
```

Cette optimisation fonctionne car les étiquettes NaN sont dans les bits de poids fort, et la valeur Integer occupe les 48 bits inférieurs — ajouter 1 au mot entier ne change que la partie valeur (tant qu'il n'y a pas de débordement dans le domaine 48 bits). Le même incrément de chemin rapide est utilisé pour les variables locales et globales dans tous les opcodes de boucle fusionnés.