# XCX Backend — Kompilator i VM

> **Pliki:** `src/backend/mod.rs`, `src/backend/vm.rs`

---

## Spis treści

1. [Kompilator bytecode](#kompilator-bytecode)
2. [Reprezentacja wartości — NaN-boxing](#reprezentacja-wartości--nan-boxing)
3. [Zestaw instrukcji (OpCodes)](#zestaw-instrukcji-opcodes)
4. [Architektura VM](#architektura-vm)
5. [Model wykonania włókien](#model-wykonania-włókien)
6. [Serwer HTTP](#serwer-http)
7. [Zarządzanie pamięcią](#zarządzanie-pamięcią)
8. [Optymalizacje pętli](#optymalizacje-pętli)

---

## Kompilator bytecode

**Plik:** `src/backend/mod.rs`

### Alokacja rejestrów

`FunctionCompiler` śledzi następny dostępny rejestr przez `next_local: usize`:

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

Zmienne lokalne (nazwane) są przypisywane do slotu przez `define_local(id, slot)` i przechowywane w `scopes: Vec<HashMap<StringId, usize>>`. Zmienne tymczasowe używają `push_reg()`/`pop_reg()` — są ponownie wykorzystywane, gdy wynik wyrażenia zostanie zużyty.

`max_locals_used` jest rejestrowane, aby `FunctionChunk::max_locals` mógł wstępnie zaalokować wektor locals dokładnie o właściwym rozmiarze przy wywołaniu funkcji.

### Deduplikacja stałych

`CompileContext::add_constant` deduplikuje stałe łańcuchowe za pomocą `string_constants: HashMap<String, usize>`. Powtarzające się stałe łańcuchowe (np. `"insert"` występujące wielokrotnie jako argument nazwy metody) ponownie używają tego samego slotu w tabeli stałych.

### Kompilacja dwuprzebiegowa

**Przebieg 1** — `register_globals_recursive`:
- Przypisuje indeks slotu każdej zmiennej globalnej i instancji deklaracji włókna
- Przypisuje indeks funkcji każdej funkcji/włóknu
- Wstępnie alokuje puste sloty `FunctionChunk` w `functions: Vec<FunctionChunk>`

**Przebieg 2** — `compile_stmt` / `compile_expr`:
- Emituje bytecode sparowany ze spanami przez `emit(op, span)`
- Instrukcje najwyższego poziomu w `main` używają `GetVar`/`SetVar` (globalne); zagnieżdżone instrukcje używają rejestrów

### FunctionChunk

```rust
pub struct FunctionChunk {
    pub bytecode:   Arc<Vec<OpCode>>,
    pub spans:      Arc<Vec<Span>>,    // spans[i] corresponds to bytecode[i]
    pub is_fiber:   bool,
    pub max_locals: usize,
}
```

Bytecode i spany są opakowane w `Arc`, aby mogły być współdzielone między wątkami roboczymi HTTP bez kopiowania.

---

## Reprezentacja wartości — NaN-boxing

Każda wartość to pojedyncze `Value(u64)` — 64-bitowe słowo. XCX używa **NaN-boxingu**: wzorzec bitowy cichego NaN IEEE 754 jest wykorzystywany jako prefiks tagu typu.

```
Bit layout: [63..52: exponent/QNAN] [51..48: type tag] [47..0: payload]

Float : stored directly as f64 bits — DOES NOT have the QNAN_BASE prefix set
Int   : QNAN_BASE | TAG_INT  | (i48 value & 0x0000_FFFF_FFFF_FFFF)
Bool  : QNAN_BASE | TAG_BOOL | (0 or 1)
Date  : QNAN_BASE | TAG_DATE | (i48 ms timestamp)
Ptr   : QNAN_BASE | TAG_XXX  | (pointer & 0x0000_FFFF_FFFF_FFFF)
```

### Stałe tagów

| Stała | Wartość | Typ |
|---|---|---|
| `QNAN_BASE` | `0x7FF0_0000_0000_0000` | bazowy marker NaN |
| `TAG_INT` | `0x0001_0000_0000_0000` | 48-bitowa liczba całkowita ze znakiem |
| `TAG_BOOL` | `0x0002_0000_0000_0000` | wartość logiczna (payload 0/1) |
| `TAG_DATE` | `0x0003_0000_0000_0000` | 48-bitowy znacznik czasu (ms) |
| `TAG_STR` | `0x0004_0000_0000_0000` | wskaźnik `Arc<Vec<u8>>` |
| `TAG_ARR` | `0x0005_0000_0000_0000` | wskaźnik `Arc<RwLock<Vec<Value>>>` |
| `TAG_SET` | `0x0006_0000_0000_0000` | wskaźnik `Arc<RwLock<SetData>>` |
| `TAG_MAP` | `0x0007_0000_0000_0000` | wskaźnik `Arc<RwLock<Vec<(Value,Value)>>>` |
| `TAG_TBL` | `0x0008_0000_0000_0000` | wskaźnik `Arc<RwLock<TableData>>` |
| `TAG_FUNC` | `0x0009_0000_0000_0000` | indeks funkcji (u32) |
| `TAG_ROW` | `0x000A_0000_0000_0000` | wskaźnik `Arc<RowRef>` |
| `TAG_JSON` | `0x000B_0000_0000_0000` | wskaźnik `Arc<RwLock<serde_json::Value>>` |
| `TAG_FIB` | `0x000C_0000_0000_0000` | wskaźnik `Arc<RwLock<FiberState>>` |
| `TAG_DB` | `0x000D_0000_0000_0000` | wskaźnik `Arc<DatabaseData>` |

Payload wskaźników używa tylko dolnych 48 bitów — ważne na wszystkich platformach x86-64 i AArch64, gdzie wskaźniki przestrzeni użytkownika mieszczą się w 48 bitach.

### Liczenie referencji

Wartości oznaczone tagiem wskaźnika niosą liczniki referencji `Arc`. VM zarządza nimi ręcznie przez `inc_ref()` / `dec_ref()` przy każdym przypisaniu, zwrocie i modyfikacji kolekcji — zapewniając zwolnienie obiektów alokowanych na stercie, gdy nie są już referencjonowane, bez garbage collectora.

---

## Zestaw instrukcji (OpCodes)

Wszystkie opcode'y są oparte na rejestrach: odwołują się do nazwanych slotów rejestrów `u8` zamiast stosu operandów.

### Ruch rejestrów / zmiennych

| OpCode | Opis |
|---|---|
| `LoadConst { dst, idx }` | Załaduj `constants[idx]` do rejestru `dst` |
| `Move { dst, src }` | Skopiuj rejestr `src` do `dst` |
| `GetVar { dst, idx }` | Załaduj `globals[idx]` do `dst` (read-lock globals) |
| `SetVar { idx, src }` | Zapisz `src` do `globals[idx]` (write-lock globals) |

### Arytmetyka

Wszystkie operacje arytmetyczne są 3-rejestrowe: `dst = src1 OP src2`. Dyspozycja typów w czasie wykonania wybiera ścieżki dla liczb całkowitych, zmiennoprzecinkowych, konkatenacji łańcuchów, arytmetyki dat lub operacji na zbiorach.

`Add`, `Sub`, `Mul`, `Div`, `Mod`, `Pow`, `IntConcat` (`++`)

### Porównania (wynik Bool)

`Equal`, `NotEqual`, `Greater`, `Less`, `GreaterEqual`, `LessEqual`

### Logika

`And { dst, src1, src2 }`, `Or { dst, src1, src2 }`, `Not { dst, src }`, `Has { dst, src1, src2 }`

### Przepływ sterowania

| OpCode | Opis |
|---|---|
| `Jump { target }` | Skok bezwarunkowy; inkrementuje `hot_counts[target]` przy skokach wstecz |
| `JumpIfFalse { src, target }` | Skok, jeśli `src` to `Bool(false)` |
| `JumpIfTrue { src, target }` | Skok, jeśli `src` to `Bool(true)` |
| `Call { dst, func_idx, base, arg_count }` | Wywołaj funkcję; wynik → `dst` |
| `Return { src }` | Zwróć wartość w `src` z bieżącej ramki |
| `ReturnVoid` | Zwróć bez wartości |
| `Halt` | Zatrzymaj wykonanie |

### Kolekcje

| OpCode | Opis |
|---|---|
| `ArrayInit { dst, base, count }` | Zbierz `count` rejestrów od `base` → nowa tablica w `dst` |
| `SetInit { dst, base, count }` | Zbierz `count` rejestrów → nowy zbiór w `dst` |
| `SetRange { dst, start, end, step, has_step }` | Zbuduj zbiór zakresu z wartości rejestrów |
| `MapInit { dst, base, count }` | Zbierz `count` par klucz-wartość → nowa mapa w `dst` |
| `TableInit { dst, skeleton_idx, base, row_count }` | Zbuduj tabelę ze stałej schematu kolumn + wartości wierszy |

### Operacje na zbiorach

`SetUnion`, `SetIntersection`, `SetDifference`, `SetSymDifference` — wszystkie 3-rejestrowe, oba operandy muszą być `TAG_SET`.

### Dyspozycja metod

| OpCode | Opis |
|---|---|
| `MethodCall { dst, kind, base, arg_count }` | Dyspozycja wbudowanej metody przez enum `MethodKind` — bez wyszukiwania łańcucha w czasie wykonania |
| `MethodCallCustom { dst, method_name_idx, base, arg_count }` | Dyspozycja dynamicznej metody (pole JSON, alias) przez łańcuch z tabeli stałych |
| `MethodCallNamed { dst, kind, base, arg_count, names_idx }` | Wywołanie metody z nazwanymi argumentami |

`base` wskazuje rejestr odbiorcy; argumenty są w `locals[base+1..base+1+arg_count]`. `MethodKind` to enum `#[derive(Copy)]` obejmujący ~50 wbudowanych metod (`Push`, `Pop`, `Get`, `Insert`, `Update`, `Delete`, `Where`, `Join`, `Sort`, `Format`, `Next`, `IsDone`, `Close` itd.).

### Operacje włókien

| OpCode | Opis |
|---|---|
| `FiberCreate { dst, func_idx, base, arg_count }` | Alokuj `FiberState`, wstępnie wypełnij locals z argumentów, zapisz `Fiber` w `dst` |
| `Yield { src }` | Zawieś włókno, zwróć wartość w `src` do wywołującego |
| `YieldVoid` | Zawieś włókno void |

### I/O i system

| OpCode | Opis |
|---|---|
| `Print { src }` | Wypisz `locals[src]` na stdout |
| `Input { dst, ty }` | Odczytaj linię ze stdin → `dst` z rzutowaniem typu |
| `Wait { src }` | Uśpij przez `src` milisekund |
| `HaltAlert { src }` | Wypisz komunikat ostrzegawczy, kontynuuj wykonanie |
| `HaltError { src }` | Wypisz błąd + info o spanie, zatrzymaj ramkę, inkrementuj error_count |
| `HaltFatal { src }` | Wypisz fatal + info o spanie, zatrzymaj ramkę, inkrementuj error_count |
| `TerminalExit` | `std::process::exit(0)` |
| `TerminalClear` | Wyczyść terminal przez escape ANSI |
| `TerminalRaw / TerminalNormal` | Włącz/wyłącz tryb surowy terminala |
| `TerminalCursor { on }` | Pokaż/ukryj kursor |
| `TerminalMove { x_src, y_src }` | Przesuń kursor |
| `TerminalWrite { src }` | Zapisz bez nowej linii |
| `InputKey { dst }` | Odczytaj klawisz (nieblokująco) |
| `InputKeyWait { dst }` | Odczytaj klawisz (blokująco) |
| `InputReady { dst }` | Sprawdź, czy dane wejściowe są dostępne |
| `EnvGet { dst, src }` | Odczytaj zmienną środowiskową |
| `EnvArgs { dst }` | Pobierz argumenty CLI |

### HTTP

| OpCode | Opis |
|---|---|
| `HttpCall { dst, method_idx, url_src, body_src }` | Proste wywołanie HTTP przez `ureq`, wynik JSON → `dst` |
| `HttpRequest { dst, arg_src }` | Pełne wywołanie HTTP z mapy konfiguracji (method, url, headers, body, timeout) → `dst` |
| `HttpRespond { status_src, body_src, headers_src }` | Wyślij odpowiedź HTTP z wewnątrz handlera włókna |
| `HttpServe { func_idx, port_src, host_src, workers_src, routes_src }` | Uruchom serwer `tiny_http`, utwórz wątki robocze |

### Magazyn

`StoreWrite`, `StoreRead`, `StoreAppend`, `StoreExists`, `StoreDelete`, `StoreList`, `StoreIsDir`, `StoreSize`, `StoreMkdir`, `StoreGlob`, `StoreZip`, `StoreUnzip`

### JSON

`JsonParse { dst, src }`, `JsonBind { idx, json_src, path_src }`, `JsonBindLocal { dst, json_src, path_src }`, `JsonInject { table_idx, json_src, mapping_src }`, `JsonInjectLocal { table_reg, json_src, mapping_src }`

### Rzutowania typów

`CastInt { dst, src }`, `CastFloat { dst, src }`, `CastString { dst, src }`, `CastBool { dst, src }`

### Kryptografia i daty

`CryptoHash`, `CryptoVerify`, `CryptoToken`, `DateNow`

### Baza danych

`DatabaseInit { dst, engine_src, path_src, tables_base_reg, table_count }`

---

## Architektura VM

**Plik:** `src/backend/vm.rs`

### Stan VM

```rust
pub struct VM {
    pub globals:     Arc<RwLock<Vec<Value>>>,
    pub error_count: AtomicUsize,
    pub traces:      Arc<RwLock<HashMap<usize, Arc<Trace>>>>,
    pub jit:         Mutex<JIT>,
}
```

`VM` jest opakowany w `Arc<VM>` i współdzielony między wątkami roboczymi HTTP. Każdy worker tworzy własny `Executor` z prywatnymi locals.

### Stan Executora

```rust
struct Executor {
    vm:               Arc<VM>,
    ctx:              SharedContext,
    current_spans:    Option<Arc<Vec<Span>>>,
    fiber_yielded:    bool,
    hot_counts:       Vec<usize>,         // IP backward jump counter
    recording_trace:  Option<Trace>,      // trace being recorded
    is_recording:     bool,
    trace_cache:      Vec<Option<Arc<Trace>>>, // compiled traces indexed by start IP
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

`SharedContext` jest tanio klonowany (dwa bump'y licznika `Arc`) i przekazywany do każdego wątku roboczego niezależnie. Bez głębokiego kopiowania.

### Przepływ wykonania

```
VM::run(main_chunk, ctx)                        [Arc<VM>]
  └─ Executor::run_frame_owned(main_chunk)
       └─ execute_bytecode_inner(bytecode, &mut ip, &mut locals, ...)
            │
            ├─ [JIT Fast Path] if trace_cache[ip].is_some():
            │     execute_trace(trace, ip, locals, globals)
            │     → returns next IP or None
            │
            └─ [Interpreter Path] fetch opcode, execute
                 ├─ Continue      → proceed normally
                 ├─ Jump(t)       → ip = t; increment hot_counts if backward
                 ├─ Return(val)   → exit frame, return val
                 ├─ Yield(val)    → suspend (fiber), return val to caller
                 └─ Halt          → stop, increment error_count
```

---

## Model wykonania włókien

Włókna to **współpracujące korutyny**, nie wątki systemu operacyjnego.

```rust
pub struct FiberState {
    pub func_id:       usize,
    pub ip:            usize,
    pub locals:        Vec<Value>,    // moved during resume, returned after
    pub is_done:       bool,
    pub yielded_value: Option<Value>, // cache for IsDone + Next pattern
}
```

### Sekwencja wznowienia (`resume_fiber`)

1. Odczytaj `func_id`, `ip` i **przenieś** `locals` z `FiberState` przez `std::mem::take` — bez klonowania
2. Uruchom `execute_bytecode` od `fiber.ip` z przeniesionymi locals
3. Przy `Yield`: ustaw `fiber_yielded = true`. Przenieś locals z powrotem do `FiberState`. Zaktualizuj `fiber.ip`. Zwróć wyprodukowaną wartość.
4. Przy `Return` / końcu bytecode: ustaw `fiber.is_done = true`. Zwróć końcową wartość.

Wznawianie/zawieszanie nie powoduje alokacji na stercie poza początkowym utworzeniem `Vec` — tylko przeniesienia.

### Wzorzec IsDone / Next

`IsDone` sprawdza `FiberState::is_done`, uwzględniając, czy `yielded_value` jest w cache. `Next` pobiera buforowaną `yielded_value`, jeśli jest obecna, lub wywołuje `resume_fiber`. Zapewnia to, że pętla `for x in fiber` nigdy nie przesuwa włókna dwukrotnie.

### Pętla For nad włóknem (`ForIterType::Fiber`)

Kompilator emituje:
1. `MethodCall(IsDone)` → `JumpIfTrue` do wyjścia
2. `MethodCall(Next)` → przypisanie do zmiennej pętli
3. Ciało pętli
4. `Jump` z powrotem do kroku 1
5. Przy `break`: `MethodCall(Close, base = fiber_reg)` oznacza włókno jako zakończone przed skokiem

---

## Serwer HTTP

`HttpServe` uruchamia serwer `tiny_http` i tworzy N wątków OS:

```rust
for _ in 0..workers {
    let server  = server.clone();    // Arc<tiny_http::Server>
    let vm      = vm_arc.clone();    // Arc<VM>
    let ctx     = self.ctx.clone();  // SharedContext (two Arc clones)
    let routes  = routes.clone();    // Arc<Vec<(String, usize)>>
    std::thread::spawn(move || { /* recv → match route → run handler fiber */ });
}
```

Każdy worker uruchamia własny `Executor` z własnymi locals. Globalne są współdzielone przez `Arc<RwLock<Vec<Value>>>`.

### Obsługa żądań

Dla każdego przychodzącego żądania worker:
1. Dopasowuje klucz `"METHOD /path"` do tabeli tras (bez rozróżniania wielkości liter)
2. Buduje obiekt JSON `{ method, url, body, ip, headers }` jako `Value::Json`
3. Przechowuje `tiny_http::Request` w `Arc<Mutex<Option<tiny_http::Request>>>` i przekazuje go do świeżego `Executor`
4. Uruchamia dopasowany handler włókna synchronicznie w `Executor` tego workera
5. Gdy handler wywołuje `net.respond(...)`, VM wykonuje `HttpRespond`, który wysyła odpowiedź
6. Jeśli handler kończy się bez `net.respond`, worker wysyła odpowiedź `500`

### Graceful shutdown

`SHUTDOWN` to `pub static AtomicBool` w `vm.rs`. Handler Ctrl+C w `main.rs` ustawia go na `true`. Wątki robocze sprawdzają go co `recv_timeout(100ms)`. Wątek główny blokuje się w pętli `sleep(500ms)` również odpytując `SHUTDOWN`. Po ustawieniu wszystkie pętle kończą się i proces kończy się czysto.

---

## Zarządzanie pamięcią

- **Bez garbage collectora**. Liczenie referencji przez `Arc` (z ręcznym `inc_ref`/`dec_ref` dla wartości wskaźnikowych NaN-boxed).
- **Wartości skalarne** (Int, Float, Bool, Date, indeks funkcji): przechowywane w całości w `u64` — zero alokacji na stercie.
- **Wartości kolekcji**: `Arc<RwLock<T>>` zapewnia współdzielone posiadanie. Klonowanie `Value` kolekcji tylko inkrementuje licznik `Arc`.
- **Mutacje**: `.insert()`, `.update()`, `.delete()` pobierają write lock. Wszystkie uchwyty do tej samej kolekcji widzą zmianę.
- **Metody tylko do odczytu**: `.size()`, `.get()`, `.contains()` pobierają read lock — dozwolonych jest wielu jednoczesnych czytelników.

---

## Optymalizacje pętli

Te opcode'y są emitowane przez kompilator, aby scalić typowe wzorce liczników pętli w pojedyncze instrukcje, redukując narzut dyspozycji i poprawiając śledzenie JIT:

| OpCode | Opis |
|---|---|
| `IncLocal { reg }` | Inkrementuj liczbę całkowitą w rejestrze `reg` o 1 |
| `IncVar { idx }` | Inkrementuj globalną pod `idx` o 1 |
| `LoopNext { reg, limit_reg, target }` | Inkrementuj `reg`, skocz do `target`, jeśli `reg <= limit_reg` |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target }` | Inkrementuj `inc_reg` (osobny licznik, np. indeks tablicy), inkrementuj `reg` (zmienna pętli), warunkowy skok |
| `IncVarLoopNext { g_idx, reg, limit_reg, target }` | Jak `IncLocalLoopNext`, ale `g_idx` to globalny licznik |
| `ArrayLoopNext { idx_reg, size_reg, target }` | Połączona iteracja indeksu tablicy |

### Transformacja optymalizacji

Kompilator sprawdza ostatnią wyemitowaną instrukcję przed końcem kroku pętli. Jeśli to `IncVar` lub `IncLocal`, zastępuje je odpowiednio `IncVarLoopNext` lub `IncLocalLoopNext`, łącząc inkrementację z warunkowym testem pętli:

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
