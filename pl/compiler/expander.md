# Expander XCX — dokumentacja

> **Plik:** `src/parser/expander.rs`  
> Uruchamiany **po** parsowaniu, **przed** analizą semantyczną.

---

## Spis treści

1. [Przegląd](#przegląd)
2. [Odpowiedzialności](#odpowiedzialności)
3. [Rozwiązywanie include](#rozwiązywanie-include)
4. [Prefiksowanie aliasów](#prefiksowanie-aliasów)
5. [Chronione nazwy](#chronione-nazwy)
6. [Kolejność wyszukiwania ścieżek include](#kolejność-wyszukiwania-ścieżek-include)

---

## Przegląd

Expander to osobny przebieg przepisywania AST po fazie parsowania, przed analizą semantyczną. Przetwarza dyrektywy `include` i `include ... as alias`.

```rust
pub struct Expander<'a> {
    interner:       &'a mut Interner,
    visiting_files: HashSet<PathBuf>,   // for circular dependency detection
    included_files: HashSet<PathBuf>,   // deduplication (each file once)
    aliases:        HashMap<StringId, String>,
    include_paths:  Vec<PathBuf>,       // additional search paths
}
```

---

## Odpowiedzialności

### 1. Rozwiązywanie include
`include "file.xcx";` zastępowane jest wstawionym AST tego pliku.

- Zależności cykliczne wykrywane przez `visiting_files: HashSet<PathBuf>`
- Pliki deduplikowane przez `included_files: HashSet<PathBuf>` (każdy plik tylko raz, chyba że z aliasem)

### 2. Prefiksowanie aliasów
`include "math.xcx" as math;` powoduje, że wszystkie nazwy najwyższego poziomu z pliku dostają prefiks `math.name`. Miejsca wywołań (`math.sin(x)`) przepisywane z `MethodCall` na `FunctionCall { name: "math.sin" }` przez `expand_expr_inplace`.

Funkcje `prefix_program()` / `prefix_stmt_impl()` / `prefix_expr_impl()` przechodzą całe pod-AST, zmieniając odwołania do symboli najwyższego poziomu.

### 3. Prefiksowanie nazw fiber
Odwołania `FiberDecl::fiber_name` też są prefiksowane, aby instancjonowanie po zmianie nazw rozwiązywało się poprawnie.

### 4. Prefiksowanie YieldFrom
Wyrażenia `StmtKind::YieldFrom` są przechodzone, aby wywołania konstruktorów fiber w `yield from` też były przemianowane.

---

## Rozwiązywanie include

```xcx
include "utils.xcx";           --- Simple include (deduplicated)
include "math.xcx" as math;    --- Aliased include
```

Po include z aliasem:
- Wszystkie symbole najwyższego poziomu z `math.xcx` dostają prefiks `math.`
- Wywołania `math.sin(x)` przepisywane z `MethodCall` na `FunctionCall { name: "math.sin" }`

Przykład:

```xcx
--- math.xcx defines:
func sin(f: x -> f) { ... }
func cos(f: x -> f) { ... }

--- After include "math.xcx" as math:
--- sin → math.sin
--- cos → math.cos
--- Call math.sin(3.14) → FunctionCall("math.sin", [3.14])
```

---

## Prefiksowanie aliasów

Algorytm `prefix_stmt_impl` / `prefix_expr_impl`:

1. Zbiera wszystkie nazwy najwyższego poziomu z programu (`top_level_names: HashSet<StringId>`)
2. Dla każdej instrukcji/wyrażenia: jeśli identyfikator należy do `top_level_names`, zastępowany jest przez `prefix.name`
3. Obsługuje w szczególności:
   - `FiberDecl::fiber_name` — aby instancjonowanie działało po zmianie nazwy
   - `StmtKind::YieldFrom` — wyrażenia fiber w `yield from`
   - `MethodCall` na obiekcie z aliasem → przepisanie na `FunctionCall`

---

## Chronione nazwy

Następujące nazwy **nigdy** nie są prefiksowane (chronione wbudowane):

```
json    date    store   halt    terminal
net     env     crypto  EMPTY   math
random  i       f       s       b
from    main
```

---

## Kolejność wyszukiwania ścieżek include

1. Względem katalogu bieżącego pliku
2. W katalogu `lib/` (względem CWD, potem w górę od ścieżki pliku wykonywalnego)

Dodatkowe ścieżki można dodać przez:

```rust
expander.add_include_path(path);
```

W `main.rs` katalog `lib/` względem CWD jest dodawany automatycznie:

```rust
if let Ok(cwd) = std::env::current_dir() {
    let lib_path = cwd.join("lib");
    if lib_path.exists() {
        expander.add_include_path(lib_path);
    }
}
```

---

## Wykrywanie błędów

| Błąd | Opis |
|---|---|
| `Circular dependency` | Wykryta pętla include (`visiting_files`) |
| `File not found` | Plik `include` nie istnieje w żadnej ze ścieżek wyszukiwania |
| `Could not read file` | Błąd I/O przy odczycie pliku |
