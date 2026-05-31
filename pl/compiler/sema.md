# Analiza semantyczna XCX (Sema) — dokumentacja

> **Pliki:** `src/sema/checker.rs`, `src/sema/symbol_table.rs`, `src/sema/interner.rs`

---

## Spis treści

1. [Przegląd](#przegląd)
2. [Interner łańcuchów](#interner-łańcuchów)
3. [Tabela symboli](#tabela-symboli)
4. [Sprawdzanie typów](#sprawdzanie-typów)
5. [Reguły kompatybilności typów](#reguły-kompatybilności-typów)
6. [Kody błędów](#kody-błędów)
7. [Raportowanie błędów](#raportowanie-błędów)

---

## Przegląd

Faza Sema waliduje AST pod kątem poprawności logicznej i spójności typów przed generowaniem bytecode. Składa się z trzech komponentów:

```
Interner → StringId (u32)
     ↓
SymbolTable → hierarchical type scopes
     ↓
Checker → collection of TypeErrors
```

Program kompilowany jest tylko gdy wynikowe `Vec<TypeError>` jest puste.

---

## Interner łańcuchów

**Plik:** `src/sema/interner.rs`

`Interner` mapuje `&str → StringId(u32)`. To jedyne źródło prawdy dla tożsamości łańcuchów w kompilatorze. Tworzony podczas lexingu/parsowania i przekazywany przez referencję do checkera i kompilatora.

```rust
Interner::intern("foo") → StringId(42)
Interner::lookup(StringId(42)) → "foo"
```

**Implementacja:**

```rust
pub struct Interner {
    map:     HashMap<String, StringId>,
    strings: Vec<String>,
}
```

Każdy unikalny łańcuch przechowywany jest raz w `strings: Vec<String>`. Reszta potoku pracuje na numerycznych ID, eliminując porównania na stercie podczas sprawdzania typów i kompilacji.

---

## Tabela symboli

**Plik:** `src/sema/symbol_table.rs`

`SymbolTable` zarządza powiązaniami zmiennych w zagnieżdżonych zakresach przez **łańcuch wskaźników rodzica**.

### Struktura

```rust
pub struct SymbolTable<'a> {
    parent: Option<&'a SymbolTable<'a>>,  // ref to surrounding scope
    scopes: Vec<HashMap<String, Type>>,   // stack of scope frames (local)
    consts: Vec<HashSet<String>>,         // which names are const, per frame
}
```

### Tworzenie zakresu potomnego

Przy wejściu w ciało funkcji lub fiber tworzona jest nowa `SymbolTable` z odwołaniem do otaczającej tabeli zamiast jej klonowania:

```rust
let mut func_symbols = SymbolTable::new_with_parent(symbols);
```

To O(1) — alokowany jest tylko jeden pusty `HashMap` dla nowej ramki. Wyszukiwania przechodzą łańcuch rodzica w razie potrzeby.

### Cykl życia zakresu

| Metoda | Opis |
|---|---|
| `enter_scope()` | Wypycha nową ramkę — dla ciał `if`, `while`, `for` |
| `exit_scope()` | Zamyka bieżący zakres |
| `new_with_parent(parent)` | Tworzy tabelę potomną — dla ciał funkcji/fiber |
| `define(name, ty, is_const)` | Zawsze zapisuje do **najbardziej wewnętrznego** zakresu |
| `lookup(name)` | Od wewnątrz na zewnątrz, potem do rodzica |
| `has_in_current_scope(name)` | Sprawdza tylko najbardziej wewnętrzną ramkę — wykrywanie redeklaracji |
| `is_const(name)` | Sprawdza, czy zakres właściciela oznaczył nazwę jako const |

### Brak zasłaniania zmiennych

XCX **nie** obsługuje zasłaniania zmiennych. Zdefiniowanie zmiennej już istniejącej w **bieżącym zakresie** zwraca `RedefinedVariable`. Zmienne z zakresów nadrzędnych są dostępne, ale nie można ich ponownie zadeklarować w zakresie potomnym pod tą samą nazwą.

---

## Sprawdzanie typów

**Plik:** `src/sema/checker.rs`

Struktura `Checker` przechodzi AST i akumuluje wartości `TypeError`.

### Stan checkera

```rust
pub struct Checker<'a> {
    interner:          &'a Interner,
    loop_depth:        usize,
    functions:         HashMap<String, FunctionSignature>,
    fiber_context:     Option<Option<Type>>,  // None=outside fiber, Some(None)=void, Some(Some(T))=typed
    is_table_lambda:   bool,
    fiber_has_yield:   bool,
    in_yield_expr:     bool,
    last_expr_was_db_io: bool,
}
```

### Flagi kontekstu

| Pole | Cel |
|---|---|
| `loop_depth: usize` | Głębokość zagnieżdżenia `while`/`for`. Zero → `break`/`continue` to błędy. Reset do 0 przy wejściu w ciało fiber. |
| `fiber_context: Option<Option<Type>>` | `None` = poza fiberem; `Some(None)` = void fiber; `Some(Some(T))` = typowany fiber zwracający `T` |
| `fiber_has_yield: bool` | Ustawiane przy `yield`. Zapisywane/przywracane przez zagnieżdżone definicje fiber. |
| `is_table_lambda: bool` | W predykatach `.where()`; gołe nazwy kolumn jako identyfikatory przez `__row_tmp` |
| `in_yield_expr: bool` | Śledzi, czy jesteśmy wewnątrz wyrażenia yield |
| `last_expr_was_db_io: bool` | Flaga operacji I/O bazy danych |

### Przebieg pre-scan

Przed sprawdzaniem ciała instrukcji checker wykonuje **skan deklaracji w przód** wszystkich węzłów `FunctionDef` i `FiberDef` w **bieżącej liście instrukcji** (bez rekurencji). Umożliwia wywołania przed definicją (wzajemna rekurencja, wywołanie przed deklaracją).

Dla każdej funkcji/fiber rejestrowane są:
- `FunctionSignature { params: Vec<Type>, return_type: Option<Type>, is_fiber: bool }` w `self.functions`
- Wpis w `SymbolTable` z `Type::Unknown` (funkcje) lub `Type::Fiber(...)` (fibery)

Wbudowane funkcje rzutowania `i`, `f`, `s`, `b` są wstępnie rejestrowane.

### Reguły inferencji typów

- Typy wyrażeń wnioskowane bottom-up z literałów i propagowane przez operatory.
- `Type::Unknown` działa jak wildcard — operacja z `Unknown` przechodzi bez błędu.
- `Type::Json` kompatybilny z dowolnym typem w przypisaniach i porównaniach.
- Promocja numeryczna: `Int op Float → Float`.
- Pusta tablica `[]` dziedziczy typ z kontekstu przypisania, jeśli dostępny.
- `Type::Table([])` (pusta lista kolumn) kompatybilna z dowolnym `Table(cols)`.

---

## Reguły kompatybilności typów

Funkcja `is_compatible(expected, actual) -> bool`:

| Reguła | Opis |
|---|---|
| Którykolwiek to `Unknown` | Zawsze kompatybilne |
| Którykolwiek to `Json` | Zawsze kompatybilne |
| `Int` ↔ `Float` | Wzajemnie kompatybilne (promocja numeryczna) |
| `Int` ↔ `Date` | Kompatybilne (znaczniki czasu to liczby całkowite) |
| `Set(X)` ↔ `Array(inner)` | Kompatybilne gdy typ elementu się zgadza |
| `Set(N)` ↔ `Set(Z)` | Kompatybilne (oba zbiory całkowitoliczbowe) |
| `Set(S)` ↔ `Set(C)` | Kompatybilne (oba zbiory łańcuchowe) |
| `Table([])` ↔ `Table(cols)` | Kompatybilne gdy któraś lista kolumn jest pusta |
| `Table(a)` ↔ `Table(b)` | Kompatybilne gdy długości się zgadzają i typy kolumn parami |
| `Map(k1,v1)` ↔ `Map(k2,v2)` | Sprawdzane rekurencyjnie |
| `Fiber(None)` ↔ `Fiber(None)` | Oba void fibery |
| `Fiber(Some(T))` ↔ `Fiber(Some(U))` | Kompatybilne jeśli T i U kompatybilne |

---

## Kody błędów

| Kod | Warunek |
|---|---|
| `[S101] UndefinedVariable(name)` | Nazwa użyta przed deklaracją |
| `[S102] RedefinedVariable(name)` | Nazwa zadeklarowana dwukrotnie w tym samym zakresie |
| `[S103] TypeMismatch { expected, actual }` | Typ wyrażenia nie pasuje do oczekiwanego |
| `[S104] InvalidBinaryOp { op, left, right }` | Operator z niekompatybilnymi typami |
| `[S105] ConstReassignment(name)` | Przypisanie do zmiennej `const` |
| `[S106] BreakOutsideLoop` | `break` poza `while`/`for` |
| `[S107] ContinueOutsideLoop` | `continue` poza `while`/`for` |
| `[S108] IndexAccessNotSupported(type)` | Indeksowanie nieobsługiwanego typu |
| `[S109] PropertyNotFound` | Właściwość nie istnieje na typie |
| `[S110] MethodNotFound` | Metoda nie istnieje na typie |
| `[S111] InvalidArgumentCount` | Nieprawidłowa liczba argumentów |
| `[S208] YieldOutsideFiber` | `yield` poza ciałem fiber |
| `[S209] FiberTypeMismatch` | `yield expr;` w void fiber (powinno być `yield;`) |
| `[S210] ReturnTypeMismatchInFiber` | Gołe `return;` w typowanym fiber (brak wartości zwracanej) |
| `[S211] CannotIterateOverVoidFiber` | Iteracja po void fiber |
| `[S212] CannotRunTypedFiber` | Wywołanie `.run()` na typowanym fiber |
| `[S301] WherePredicateNameCollision` | Nazwa zmiennej lokalnej koliduje z nazwą kolumny w `.where()` |
| `[S302] TableRowCountMismatch` | Wiersz tabeli ma inną liczbę kolumn niż schemat |
| `[D401] Rule violation` | `remove()` bez `.where()` |
| `Other(msg)` | Różne błędy kontekstowe (liczba argumentów, nieznana metoda itd.) |

---

## Raportowanie błędów

Wartości `TypeError` niosą `Span { line, col, len }`. Po sprawdzeniu `main.rs` przekazuje każdy błąd do `Reporter::error()`, który drukuje:

```
ERROR: [S101] Undefined variable: foo
   12 | i: x = foo + 1;
              ~~~
```

Kompilacja zatrzymuje się zaraz po raportowaniu błędów — bytecode nie jest generowany, jeśli istnieje jakikolwiek `TypeError`.

---

## Sprawdzanie wywołań funkcji

Dla nazwanych wywołań funkcji (`ExprKind::FunctionCall`):
1. Najpierw wyszukiwanie w `self.functions` (zadeklarowane funkcje/fibery)
2. Jeśli brak — w tabeli symboli (wartości funkcji pierwszej klasy, wywołania dynamiczne)
3. Jeśli sygnatura to fiber, wywołanie zwraca `Type::Fiber(Some(ret))` (instancjonowanie fiber, nie bezpośrednie wywołanie)
4. Dodatkowe argumenty ponad zadeklarowaną liczbę parametrów są sprawdzane, ale nie odrzucane (tolerancja wariadyczna)
5. Nierozwiązane wywołania dodają błąd `UndefinedVariable`

---

## Sprawdzanie `.where()` na tabelach

Przy `table.where(predicate)`:
1. Otwierany jest tymczasowy zakres i definiowane `__row_tmp: Table(cols)`
2. `is_table_lambda` ustawiane na `true`
3. `collect_pred_idents()` zbiera wszystkie nazwy `Identifier` w predykacie
4. Jeśli identyfikator istnieje w zewnętrznym zakresie **i** pasuje do nazwy kolumny → `S301 WherePredicateNameCollision`
5. Predykat sprawdzany typowo; musi zwracać `Bool`
6. Zakres zamykany, `is_table_lambda` przywracane
