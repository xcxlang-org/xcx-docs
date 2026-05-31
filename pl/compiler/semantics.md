# Analiza semantyczna XCX (Sema) — v3.0

Faza Sema waliduje AST pod kątem poprawności logicznej i spójności typów przed generowaniem bytecode. Składa się z dwóch komponentów: **tabeli symboli** i **sprawdzania typów**.

## Interner łańcuchów (`src/sema/interner.rs`)

`Interner` mapuje `&str → StringId(u32)`. To jedyne źródło prawdy dla tożsamości łańcuchów w kompilatorze. Tworzony podczas lexingu/parsowania, przekazywany przez referencję do checkera i kompilatora.

```
Interner::intern("foo") → StringId(42)
Interner::lookup(StringId(42)) → "foo"
```

## Tabela symboli (`src/sema/symbol_table.rs`)

`SymbolTable` zarządza powiązaniami zmiennych w zagnieżdżonych zakresach przez **łańcuch wskaźników rodzica**.

### Struktura

```rust
pub struct SymbolTable<'a> {
    parent: Option<&'a SymbolTable<'a>>,  // reference to enclosing scope
    scopes: Vec<HashMap<String, Type>>,   // stack of scope frames (local to this table)
    consts: Vec<HashSet<String>>,         // which names are const, per scope frame
}
```

### Tworzenie zakresu potomnego

Przy wejściu w ciało funkcji lub fiber tworzona jest nowa `SymbolTable` z odwołaniem do otaczającej tabeli zamiast jej klonowania:

```rust
let mut func_symbols = SymbolTable::new_with_parent(symbols);
```

To O(1) — alokowany jest tylko jeden pusty `HashMap` dla nowej ramki. Wyszukiwanie idzie w górę łańcucha rodzica w razie potrzeby.

### Cykl życia zakresu

- `enter_scope()` / `exit_scope()` — push/pop lokalnych ramek w tabeli — dla ciał `if`, `while`, `for`.
- `new_with_parent(parent)` — tworzy tabelę potomną powiązaną z rodzicem — dla ciał funkcji i fiber.
- `define(name, ty, is_const)` — zawsze zapisuje do **najbardziej wewnętrznego** (bieżącego) zakresu bieżącej tabeli.
- `lookup(name)` — przechodzi lokalne zakresy od wewnątrz na zewnątrz, potem łańcuch `parent`.
- `has_in_current_scope(name)` — sprawdza tylko najbardziej wewnętrzną ramkę bieżącej tabeli — do wykrywania redeklaracji w tym samym bloku.
- `is_const(name)` — znajduje zakres właściciela `name`, potem sprawdza `consts[scope_index]`.

### Ważne: brak zasłaniania zmiennych

XCX **nie** obsługuje zasłaniania zmiennych. Zdefiniowanie zmiennej już istniejącej w **bieżącym zakresie** zgłasza `RedefinedVariable`. Zmienne z zakresów nadrzędnych są dostępne, ale nie można ich ponownie zadeklarować w zakresie potomnym pod tą samą nazwą.

## Sprawdzanie typów (`src/sema/checker.rs`)

Struktura `Checker` przechodzi AST i akumuluje wartości `TypeError`. Program kompilowany jest tylko gdy wynikowe `Vec<TypeError>` jest puste.

### Stan checkera

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

### Przebieg pre-scan (`pre_scan_stmts`)

Przed sprawdzaniem ciała instrukcji checker wykonuje **skan deklaracji w przód** wszystkich węzłów `FunctionDef` i `FiberDef` w **bieżącej liście instrukcji** (bez rekurencji). Umożliwia wywołania funkcji i fiber przed ich definicją w pliku (wzajemna rekurencja, wywołanie przed deklaracją).

Dla każdej funkcji/fiber rejestrowane są:
- `FunctionSignature { params: Vec<Type>, return_type: Option<Type>, is_fiber: bool }` w `self.functions`
- Wpis w `SymbolTable` z `Type::Unknown` (funkcje) lub `Type::Fiber(...)` (fibery)

Wbudowane funkcje rzutowania `i`, `f`, `s`, `b` są wstępnie rejestrowane z parametrem `Type::Unknown` i odpowiednim typem zwracanym.

### Flagi kontekstu

| Pole | Cel |
|---|---|
| `loop_depth: usize` | Głębokość zagnieżdżenia `while`/`for`. Zero → `break`/`continue` to błędy. Reset do 0 przy wejściu w ciało fiber. |
| `fiber_context: Option<Option<Type>>` | `None` = poza fiberem; `Some(None)` = void fiber; `Some(Some(T))` = typowany fiber zwracający `T`. |
| `fiber_has_yield: bool` | Ustawiane przy `yield`. Zapisywane/przywracane w zagnieżdżonych definicjach fiber. |
| `is_table_lambda: bool` | W predykatach `.where()`; pozwala na gołe nazwy kolumn jako identyfikatory przez `__row_tmp`. |

### Reguły inferencji typów

- Typy wyrażeń wnioskowane bottom-up z literałów i propagowane przez operatory.
- `Type::Unknown` działa jak wildcard — operacja z `Unknown` przechodzi bez błędu.
- `Type::Json` kompatybilny z dowolnym typem w przypisaniach i porównaniach.
- Promocja numeryczna: `Int op Float → Float`.
- Pusta tablica `[]` dziedziczy typ z kontekstu przypisania, jeśli dostępny.
- `Type::Table([])` (pusta lista kolumn) kompatybilna z dowolnym `Table(cols)` — informacja o kolumnach propagowana z powrotem do zadeklarowanego typu zmiennej po inferencji.

### `is_compatible(expected, actual) -> bool`

Kluczowe reguły kompatybilności:

| Reguła | Opis |
|---|---|
| Którykolwiek to `Unknown` | Zawsze kompatybilne |
| Którykolwiek to `Json` | Zawsze kompatybilne |
| `Int` ↔ `Float` | Wzajemnie kompatybilne (promocja numeryczna) |
| `Int` ↔ `Date` | Kompatybilne (znaczniki czasu to liczby całkowite) |
| `Set(X)` ↔ `Array(inner)` | Kompatybilne gdy typ elementu wewnętrznego się zgadza |
| `Set(N)` ↔ `Set(Z)` | Kompatybilne (oba zbiory całkowitoliczbowe) |
| `Set(S)` ↔ `Set(C)` | Kompatybilne (oba zbiory łańcuchowe) |
| `Table([])` ↔ `Table(cols)` | Kompatybilne gdy któraś lista kolumn jest pusta |
| `Table(a)` ↔ `Table(b)` | Kompatybilne gdy długości się zgadzają i wszystkie typy kolumn parami kompatybilne |
| `Map(k1,v1)` ↔ `Map(k2,v2)` | Sprawdzane rekurencyjnie |
| `Fiber(None)` ↔ `Fiber(None)` | Oba void fibery |
| `Fiber(Some(T))` ↔ `Fiber(Some(U))` | Kompatybilne iff T i U kompatybilne |

### Sprawdzanie wywołań funkcji

Dla nazwanych wywołań funkcji (`ExprKind::FunctionCall`):
1. Wyszukiwanie w `self.functions` (zadeklarowane funkcje/fibery).
2. Jeśli brak — w tabeli symboli (wartości funkcji pierwszej klasy, wywołania dynamiczne).
3. Jeśli sygnatura to fiber, wywołanie zwraca `Type::Fiber(Some(ret))` (instancjonowanie fiber, nie bezpośrednie wywołanie).
4. Dodatkowe argumenty ponad zadeklarowaną liczbę parametrów są sprawdzane, ale nie odrzucane (tolerancja wariadyczna).
5. Nierozwiązane wywołania dodają `UndefinedVariable`.

Dla wywołań na poziomie instrukcji (`StmtKind::FunctionCallStmt`):
- Niezgodna liczba argumentów → `Other("Function expects N arguments, got M")`.
- Nieznana nazwa poza tabelą symboli → `UndefinedVariable`.

### Sprawdzanie deklaracji fiber (`FiberDecl`)

1. Wyszukiwanie `fiber_name` w `self.functions` (preferowane) lub tabeli symboli.
2. Weryfikacja, że sygnatura ma `is_fiber: true`.
3. Sprawdzenie typów argumentów względem parametrów.
4. Definicja nowej zmiennej z `Type::Fiber(inner_type)`.

### Sprawdzanie `.where()` na tabelach

Przy `table.where(predicate)`:
1. Otwierany jest tymczasowy zakres i definiowane `__row_tmp: Table(cols)`.
2. `is_table_lambda` ustawiane na `true`.
3. `collect_pred_idents()` zbiera wszystkie nazwy `Identifier` w predykacie.
4. Jeśli identyfikator istnieje w zewnętrznym zakresie **i** pasuje do nazwy kolumny → `S301 WherePredicateNameCollision`.
5. Predykat sprawdzany typowo; musi zwracać `Bool`.
6. Zakres zamykany, `is_table_lambda` przywracane.

### Sprawdzanie `Table.join()`

Checker scala definicje kolumn: kolumny lewej tabeli plus kolumny prawej nieobecne po lewej (równość `StringId`). Przy konflikcie nazwy kolumny zachowywana jest z prawej tabeli. Wynik: `Type::Table(combined_cols)`.

### Sprawdzanie pętli `for`

Pole `iter_type` w `ForIterType` jest modyfikowane podczas sprawdzania na podstawie typu `start`:
- `Type::Array(_)` → `ForIterType::Array`, zmienna pętli dostaje typ wewnętrzny
- `Type::Set(st)` → `ForIterType::Set`, zmienna pętli dostaje typ elementu zbioru
- `Type::Table(cols)` → traktowane jak `ForIterType::Array`, zmienna pętli dostaje `Type::Table(cols)`
- `Type::Fiber(inner)` → `ForIterType::Fiber`, zmienna pętli dostaje wewnętrzny typ yield
- `Type::Int` (ze słowem `to`) → pozostaje `ForIterType::Range`, zmienna pętli to `Int`

---

## Walidowane kody błędów

| Kod | Warunek |
|---|---|
| `UndefinedVariable(name)` | Nazwa użyta przed deklaracją |
| `RedefinedVariable(name)` | Nazwa zadeklarowana dwukrotnie w tym samym zakresie |
| `ConstReassignment(name)` | Przypisanie do zmiennej `const` |
| `TypeMismatch { expected, actual }` | Typ wyrażenia nie pasuje do oczekiwanego |
| `InvalidBinaryOp { op, left, right }` | Operator z niekompatybilnymi typami |
| `BreakOutsideLoop` | `break` poza `while`/`for` |
| `ContinueOutsideLoop` | `continue` poza `while`/`for` |
| `[S208] YieldOutsideFiber` | `yield` poza ciałem fiber |
| `[S209] FiberTypeMismatch` | `yield expr;` w void fiber (powinno być `yield;`) |
| `[S210] ReturnTypeMismatchInFiber` | Gołe `return;` w typowanym fiber (brak wartości zwracanej) |
| `[S301] WherePredicateNameCollision` | Nazwa zmiennej lokalnej koliduje z nazwą kolumny w `.where()` |
| `Other(msg)` | Różne błędy kontekstowe (liczba argumentów, nieznana metoda, iteracja po void fiber itd.) |

---

## Raportowanie błędów

Wartości `TypeError` niosą `Span { line, col, len }`. Po sprawdzeniu `main.rs` przekazuje każdy błąd do `Reporter::error()`, który drukuje:
1. `ERROR: <message>`
2. Odpowiednią linię źródła z numerem linii
3. Podkreślenie `~~~` od `col` o długości `len`

Kompilacja zatrzymuje się zaraz po raportowaniu błędów — bytecode nie jest generowany, jeśli istnieje jakikolwiek `TypeError`.
