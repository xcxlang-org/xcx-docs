# Kompilator XCX — dokumentacja v3.0

> Kompilator języka XCX zaimplementowany w Rust. Wieloetapowy potok: lexer → parser → analiza semantyczna → kompilator bytecode → maszyna wirtualna → JIT (Cranelift).

---

## Spis treści

1. [Przegląd architektury](#przegląd-architektury)
2. [Szybki start](#szybki-start)
3. [Struktura projektu](#struktura-projektu)
4. [Potok kompilacji](#potok-kompilacji)
5. [Moduły](#moduły)
   - [Lexer (Scanner)](#lexer-scanner)
   - [Parser (Pratt)](#parser-pratt)
   - [Expander](#expander)
   - [Analiza semantyczna (Sema)](#analiza-semantyczna-sema)
   - [Kompilator (Backend)](#kompilator-backend)
   - [Maszyna wirtualna (VM)](#maszyna-wirtualna-vm)
   - [JIT (Cranelift)](#jit-cranelift)
6. [Kluczowe decyzje projektowe](#kluczowe-decyzje-projektowe)
7. [System diagnostyki](#system-diagnostyki)
8. [Bezpieczeństwo](#bezpieczeństwo)

---

## Przegląd architektury

```
Source Code (.xcx)
        │
        ▼
  1. Lexer         src/lexer/scanner.rs       → token stream
        │
        ▼
  2. Parser        src/parser/pratt.rs        → raw AST (Program)
        │
        ▼
  3. Expander      src/parser/expander.rs     → expanded AST (include, aliases)
        │
        ▼
  4. Sema          src/sema/checker.rs        → validated, annotated AST
        │
        ▼
  5. Compiler    src/backend/mod.rs         → FunctionChunk + constants + functions
        │
        ▼
  6. VM            src/backend/vm.rs          → bytecode execution (register-based)
        │  hot loops → trace recording
        ▼
  7. JIT           src/backend/jit.rs         → native machine code (Cranelift)
```

---

## Szybki start

```bash
# Uruchom REPL
xcx

# Uruchom plik
xcx program.xcx

# Wersja
xcx --version

# Pomoc
xcx --help
```

**W REPL:**

```
xcx> !help     # wyświetl pomoc
xcx> !clear    # wyczyść ekran
xcx> !exit     # wyjście
```

---

## Struktura projektu

```
src/
├── lexer/
│   ├── scanner.rs        # Skaner bajtów (&[u8])
│   └── token.rs          # TokenKind i Span
├── parser/
│   ├── pratt.rs          # Parser Pratt (tokeny → AST)
│   ├── expander.rs       # Rozwiązywanie include i prefiksowanie aliasów
│   └── ast.rs            # Definicje węzłów AST (Expr, Stmt, Type, Program)
├── sema/
│   ├── checker.rs        # Sprawdzanie typów i rozwiązywanie zmiennych
│   ├── symbol_table.rs   # Hierarchiczna tabela symboli
│   └── interner.rs       # Interner łańcuchów (str → StringId)
├── backend/
│   ├── mod.rs            # Kompilator bytecode (AST → OpCode)
│   ├── vm.rs             # VM oparta na rejestrach z NaN-boxing + haki JIT
│   ├── jit.rs            # Kompilator śladów Cranelift
│   └── repl.rs           # Interaktywny REPL
└── diagnostic/
    └── report.rs         # Reporter błędów z podświetlaniem kodu
```

---

## Potok kompilacji

### 1. Lexer
Konwertuje bajty źródła na tokeny. Działa na `&[u8]` — bez alokacji `Vec<char>`.

### 2. Parser
Parser Pratt (Top-Down Operator Precedence) buduje AST. Jednotokenowy lookahead. Odzyskiwanie po błędach przez `synchronize()`.

### 3. Expander
Uruchamiany **po** parsowaniu, **przed** analizą semantyczną. Rozwiązuje dyrektywy `include` i prefiksuje nazwy aliasów.

### 4. Analiza semantyczna
Sprawdza typy, wykrywa niezdefiniowane zmienne, waliduje kontekst fiber/pętli. Zbiera wszystkie błędy przed generowaniem bytecode.

### 5. Kompilator
Dwa przebiegi. Przebieg 1: rejestruje globalne/funkcje. Przebieg 2: emituje bytecode z adnotacjami Span.

### 6. VM
Maszyna wirtualna oparta na rejestrach. Każda ramka ma płaski `Vec<Value>` indeksowany numerami slotów `u8`. Wartości 8-bajtowe (NaN-boxing).

### 7. JIT
Automatyczna kompilacja gorących pętli do kodu natywnego przez Cranelift. Przezroczyste dla programisty.

---

## Moduły

Szczegółowe opisy w osobnych plikach:

- [`lexer.md`](lexer.md) — Lexer / Scanner
- [`parser.md`](parser.md) — Parser Pratt i AST
- [`expander.md`](expander.md) — Expander
- [`sema.md`](sema.md) — Analiza semantyczna
- [`backend.md`](backend.md) — Kompilator i VM
- [`jit.md`](jit.md) — JIT Cranelift
- [`language.md`](language.md) — Referencja języka XCX

---

## Kluczowe decyzje projektowe

### NaN-boxing wartości
Każda wartość to pojedyncze `u64` (8 bajtów). Wysokie bity cichego NaN IEEE 754 służą jako tagi typu. Dolne 48 bitów niosą payload. Brak alokacji na stercie dla skalarów, brak narzutu tagów w enumach, brak pośrednictwa wskaźników dla typów prostych.

### VM oparta na rejestrach
Zamiast stosu operandów — płaska tablica `Vec<Value>` na ramkę. OpCode odwołują się bezpośrednio do rejestrów źródłowych/docelowych. Prosta alokacja rejestrów w kompilatorze metodą bump-pointer.

### Tracing JIT (Cranelift)
Skoki wsteczne (krawędzie pętli) są liczone per IP. Po osiągnięciu progu (50 iteracji) rozpoczyna się nagrywanie śladu. Ukończony ślad jest kompilowany do kodu natywnego przez Cranelift. Strażniki typów (`GuardInt`, `GuardFloat`) obsługują specjalizację typów; nieudana straż powoduje powrót do interpretera.

### Internowanie łańcuchów
Wszystkie identyfikatory i literały łańcuchowe są internowane przez `Interner` do `StringId (u32)`. Eliminuje powtarzalne alokacje łańcuchów i porównania na stercie w całym potoku.

### Kompilacja dwuprzebiegowa
Przebieg 1 (`register_globals_recursive`) przypisuje indeksy wszystkim globalnym i funkcjom przed emisją bytecode — umożliwia wzajemną rekurencję i wywołania przed deklaracją.

---

## System diagnostyki

`Reporter` w `src/diagnostic/report.rs` produkuje kontekstowe komunikaty błędów:

```
ERROR: [S101] Undefined variable: foo
   12 | i: x = foo + 1;
              ~~~
```

Każdy błąd zawiera:
- **Level**: ERROR lub HALT
- **Location**: numer linii i kolumny
- **Visual highlighting**: odpowiednią linię źródła z podkreśleniem `~~~`

Błędy semantyczne (`TypeError`) są zbierane w `Vec` podczas fazy sprawdzania i raportowane naraz przed generowaniem bytecode.

---

## Bezpieczeństwo

### Ochrona przed SSRF w sieci
Zablokowane cele:
- URL `file://`
- `169.254.x.x` (link-local / endpoint metadanych AWS)
- Zakresy prywatne: `10.x`, `192.168.x`, `172.16–31.x` (z wyłączeniem localhost)

Stosowane zarówno do `HttpCall`, jak i `HttpRequest`.

### Limit rozmiaru treści HTTP
W `HttpServe` treść odpowiedzi jest sprawdzana po `into_string()`. Jeśli przekracza **10 MB**, zwracany jest błąd JSON 413 zamiast rzeczywistej treści.

### Nagłówki CORS
Wszystkie odpowiedzi `HttpServe` automatycznie zawierają:
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

Żądania preflight `OPTIONS` otrzymują odpowiedź `204` bez wywoływania handlera.

### Bezpieczne ścieżki plików
Operacje `store.*` walidują ścieżki — blokują `..`, ścieżki bezwzględne i litery dysków Windows.
