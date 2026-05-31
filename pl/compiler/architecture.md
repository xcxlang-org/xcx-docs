# Architektura kompilatora XCX — v3.0

Kompilator XCX jest zaimplementowany w Rust i stosuje architekturę wieloetapowego potoku.

## Potok kompilacji

```
Source Code
    │
    ▼
1. Lexer (Scanner)        — src/lexer/scanner.rs
    │  Produces: Token stream
    ▼
2. Parser (Pratt)         — src/parser/pratt.rs
    │  Produces: Raw AST (Program)
    ▼
3. Expander               — src/parser/expander.rs
    │  Produces: Expanded AST (include directives resolved, aliases prefixed)
    ▼
4. Type Checker (Sema)    — src/sema/checker.rs
    │  Produces: Validated, annotated AST
    ▼
5. Compiler (Backend)     — src/backend/mod.rs
    │  Produces: FunctionChunk (main) + Arc<Vec<Value>> constants + Arc<Vec<FunctionChunk>> functions
    ▼
6. VM                     — src/backend/vm.rs
    │  Executes register-based bytecode
    │  Hot loops detected → Trace recording begins
    ▼
7. JIT (Cranelift)        — src/backend/jit.rs
       Compiles recorded traces to native machine code
```

> **Uwaga**: Expander jest częścią modułu `src/parser/`, ale działa jako odrębna faza po parsowaniu, przed analizą semantyczną. JIT to opcjonalna warstwa przyspieszenia aktywowana automatycznie dla gorących pętli — wykonywanie bytecode trwa bez przerwy, jeśli ślad nie został jeszcze skompilowany.

## Struktura projektu

```
src/
├── lexer/
│   ├── scanner.rs      # Skaner na poziomie bajtów (&[u8])
│   └── token.rs        # Definicje TokenKind i Span
├── parser/
│   ├── pratt.rs        # Parser Pratt (strumień tokenów → AST)
│   ├── expander.rs     # Rozwiązywanie include i prefiksowanie aliasów
│   └── ast.rs          # Definicje węzłów AST (Expr, Stmt, Type, Program)
├── sema/
│   ├── checker.rs      # Sprawdzanie typów i rozwiązywanie zmiennych
│   ├── symbol_table.rs # Hierarchiczna tabela zakresów/symboli z łańcuchem wskaźników rodzica
│   └── interner.rs     # Interner łańcuchów (str → StringId)
├── backend/
│   ├── mod.rs          # Kompilator bytecode (AST → OpCode) z alokatorem rejestrów
│   ├── vm.rs           # VM oparta na rejestrach z wartościami NaN-boxed + haki tracing JIT
│   ├── jit.rs          # Kompilator śladów oparty na Cranelift
│   └── repl.rs         # Interaktywny REPL
└── diagnostic/
    └── report.rs       # Reporter błędów z podświetlaniem źródła
```

## System diagnostyki

Kompilator używa struktury `Reporter` do produkcji kontekstowych komunikatów błędów. Każdy błąd zawiera:
- **Level**: wariant ERROR lub HALT
- **Location**: numer linii i kolumny
- **Visual highlight**: odpowiednią linię źródła z podkreśleniem `~~~`

Błędy semantyczne (`TypeError`) są zbierane w `Vec` podczas fazy sprawdzania i raportowane naraz przed rozpoczęciem generowania bytecode. Jeśli występują błędy, kompilacja zatrzymuje się w tym momencie.

Błędy wykonania produkowane przez VM zawierają informację o lokalizacji w źródle (linia i kolumna) pochodzącą z tabeli `spans` przechowywanej obok bytecode każdego `FunctionChunk`.

## Kluczowe decyzje projektowe

### Wartości NaN-boxed
Wszystkie wartości reprezentowane są jako pojedyncze `u64` z użyciem NaN-boxingu. Wysokie bity cichego NaN IEEE 754 służą jako tagi typu, a dolne 48 bitów niosą payload (liczba całkowita, wartość logiczna lub wskaźnik). Liczby zmiennoprzecinkowe są przechowywane bez zmian. Każda `Value` ma dokładnie 8 bajtów — brak alokacji na stercie dla skalarów, brak narzutu tag-on-enum, brak pośrednictwa wskaźników dla typów prostych. Pełny układ tagów: `src/backend/vm.rs`.

### VM oparta na rejestrach
VM jest **oparta na rejestrach**, nie na stosie. Każda ramka funkcji posiada płaski `Vec<Value>` indeksowany numerem slotu. OpCode odwołują się bezpośrednio do rejestrów źródłowych i docelowych (pola `dst`, `src1`, `src2`). `FunctionCompiler` utrzymuje prosty alokator bump (`push_reg()` / `pop_reg()`) dla rejestrów tymczasowych. Zmienne lokalne i tymczasowe żyją w tej samej płaskiej tablicy — nie ma osobnego stosu operandów.

### Tracing JIT (Cranelift)
Gorące skoki wsteczne wykrywane są przez licznik `hot_counts: Vec<usize>` na ramkę. Gdy krawędź zwrotna pętli osiąga próg (50 iteracji), rozpoczyna się faza nagrywania śladu. `Executor` rejestruje warianty `TraceOp` — typowane, wyspecjalizowane wersje operacji interpretera — aż ślad się zamknie (pętla z powrotem do początkowego IP). Ukończony `Trace` jest kompilowany do kodu natywnego przez Cranelift i buforowany w `trace_cache`. Kolejne iteracje tej pętli omijają interpreter. Strażniki (`GuardInt`, `GuardFloat`, `GuardTrue`, `GuardFalse`) w śladzie obsługują specjalizację typów; niepowodzenie strażnika powoduje side-exit z powrotem do interpretera pod właściwym IP.

### Internowanie łańcuchów
Wszystkie identyfikatory i literały łańcuchowe są internowane przez `Interner` do `StringId` (u32). Unika to powtarzalnych alokacji łańcuchów i porównań na stercie w całym potoku.

### Deduplikacja stałych
Kompilator utrzymuje `string_constants: HashMap<String, usize>`, które zapewnia, że każda unikalna wartość łańcuchowa jest przechowywana tylko raz w tabeli stałych. Szczególnie skuteczne dla wbudowanych nazw metod emitowanych często podczas kompilacji.

### Dyspozycja metod przez enum
Wywołania wbudowanych metod kompilowane są do `OpCode::MethodCall { kind: MethodKind, base, arg_count }`, gdzie `MethodKind` to enum `Copy` obejmujący ~50 wbudowanych metod. Nieznane lub dynamiczne nazwy metod (np. dostęp do pól JSON) używają osobnej ścieżki `OpCode::MethodCallCustom { method_name_idx, base, arg_count }`. Eliminuje to porównania łańcuchów w pętli dyspozycji VM.

### Kompilacja dwuprzebiegowa
Backend wykonuje pierwszy przebieg (`register_globals_recursive`), aby przypisać indeksy wszystkim globalnym, funkcjom i fiberom przed emisją bytecode.

### Bytecode z adnotacją Span
Każdy emitowany opcode jest sparowany ze `Span` z AST źródłowego. `FunctionChunk` przechowuje `spans: Arc<Vec<Span>>` obok `bytecode: Arc<Vec<OpCode>>`. VM używa tego do precyzyjnych komunikatów błędów wykonania.

### Fiber jako korutyna
Fibery nie są wątkami OS. Każdy `FiberState` przechowuje własne `ip` i `locals`, wznawiane kooperatywnie przez VM. Przy yield zmienne lokalne wracają do `FiberState`; przy resume są z niego ponownie wyciągane. Bez alokacji poza początkowym utworzeniem `Vec`.

### Reprezentacja Value bezpieczna dla wątków
Wszystkie współdzielone mutowalne kolekcje używają `Arc<RwLock<T>>` (przez `parking_lot`). Bytecode i spans `FunctionChunk` opakowane są w `Arc<Vec<...>>`, aby można je było współdzielić między wątkami roboczymi HTTP bez kopiowania. `Value` jest `Copy` + `Send` + `Sync`.

### Graceful shutdown HTTP
Globalny `AtomicBool` (`SHUTDOWN`) ustawiany jest przez handler sygnału Ctrl+C (przez crate `ctrlc`). Wszystkie wątki robocze HTTP i główna pętla dyspozycji odpytują tę flagę i kończą się czysto przed zakończeniem procesu.

### Tabela symboli z łańcuchem zakresów
`SymbolTable` używa łańcucha powiązanych wskaźnikami rodzica zamiast głębokiego klonowania. Wejście w zakres funkcji tworzy nową `SymbolTable` z odwołaniem do rodzica — wyszukiwanie idzie w górę łańcucha. Wejście w zakres funkcji to O(1) zamiast O(n).

### Skaner na poziomie bajtów
Lexer działa na `&[u8]` (odwołanie do oryginalnych bajtów źródła) bez alokacji `Vec<char>`. Obsługa Unicode tylko tam, gdzie potrzeba. Wykrywanie komentarzy używa `slice.starts_with(b"---")`.
