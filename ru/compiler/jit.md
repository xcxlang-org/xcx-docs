# Трассовый JIT XCX (Cranelift) — документация

> **Файл:** `src/backend/jit.rs`  
> XCX 3.1 включает **духрежимный JIT**, автоматически компилирующий горячие циклы и часто вызываемые функции в машинный код через [Cranelift](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift).

---

## Содержание

1. [Обзор](#обзор)
2. [Два режима JIT](#два-режима-jit)
3. [Tracing JIT — обнаружение горячих циклов](#tracing-jit--обнаружение-горячих-циклов)
4. [Запись трассы](#запись-трассы)
5. [Компиляция трассы](#компиляция-трассы)
6. [Выполнение трассы](#выполнение-трассы)
7. [Method JIT — компиляция функций](#method-jit--компиляция-функций)
8. [Варианты TraceOp](#варианты-traceop)
9. [Экспортируемые функции C](#экспортируемые-функции-c)
10. [Конфигурация JIT](#конфигурация-jit)

---

## Обзор

JIT **полностью прозрачен** для разработчика — включается автоматически и возвращается в интерпретатор при срабатывании стражей типов или неподдерживаемых операций.

Существуют два независимых пути компиляции:

```
[Tracing JIT] — for hot loops
  Interpreter executes loop
      ↓ (50 iterations)
  Trace recording starts
      ↓ (loop returns to start IP)
  Trace compiled by Cranelift → native C function (3-arg signature)
      ↓ (next iteration)
  Native code executed directly (interpreter bypass)
      ↓ (failed guard)
  Fallback to interpreter at the correct IP

[Method JIT] — for frequently-called functions without loops
  Function is called
      ↓ (10 calls)
  Entire function bytecode compiled by Cranelift → native C function (5-arg signature)
      ↓ (next call)
  Native code called directly (bypasses interpreter + tracing machinery)
```

---

## Два режима JIT

### JITFunction (Tracing)

```rust
pub type JITFunction = unsafe extern "C" fn(
    *mut VMValue,   // locals_ptr
    *mut VMValue,   // globals_ptr
    *const VMValue, // consts_ptr
) -> i32            // next_ip (0 = continue, >0 = side-exit IP, <0 = halt)
```

Używany dla trace gorących pętli. Wartość zwracana to następny wskaźnik instrukcji.

### MethodJitFunction (Method)

```rust
pub type MethodJitFunction = unsafe extern "C" fn(
    *mut VMValue,   // locals_ptr
    *mut VMValue,   // globals_ptr
    *const VMValue, // consts_ptr
    *mut VM,        // vm_ptr
    *mut Executor,  // executor_ptr
) -> u64            // return value as raw Value bits
```

Używany dla skompilowanych pełnych funkcji. Zwraca wartość zwracaną funkcji bezpośrednio jako surowe bity NaN-boxed. Wymaga dostępu do VM i Executora dla wywołań rekurencyjnych i dyspozycji metod.

---

## Tracing JIT — обнаружение горячих циклов

Przy każdym `Jump { target }`, gdzie `target < current_ip` (skok wstecz, tj. krawędź powrotu pętli):

1. `hot_counts[target]` jest inkrementowane
2. Gdy `hot_counts[target] >= 50`, rozpoczyna się nowy `Trace` z `start_ip = target`
3. Nagrywanie trace jest chronione: zaczyna się tylko, gdy `trace_cache[target].is_none()` (brak skompilowanego trace) i `is_recording` jest false

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

Próg wynosi **50** dla głównego opcode `Jump` i **1000** dla scalonych opcode pętli (`LoopNext`, `IncVarLoopNext`, `IncLocalLoopNext`).

---

## Запись трассы

Gdy `is_recording` jest true, interpreter wykonuje każdy opcode normalnie **i** rejestruje `TraceOp` wyspecjalizowany dla bieżących typów w czasie wykonania. Przykłady:

- `Add` na dwóch rejestrach całkowitych rejestruje `GuardInt { reg: src1 }`, `GuardInt { reg: src2 }`, `AddInt { dst, src1, src2 }` — nie generyczne `Add`
- `JumpIfFalse`, który **nie** jest wykonany, rejestruje `GuardTrue { reg: src, fail_ip: target }` — twierdząc, że gałąź jest zawsze false
- `JumpIfFalse`, który **jest** wykonany, rejestruje `GuardFalse { reg: src, fail_ip: next_ip }`

Jeśli opcode nie może być śledzony (złożone wywołania metod, operacje na łańcuchach itd.), nagrywanie jest przerywane i `is_recording` jest ustawiane z powrotem na false.

### Co można śledzić?

| Możliwe do śledzenia | Niemożliwe do śledzenia |
|---|---|
| Arytmetyka Int/Float | Metody obiektów (oprócz ArraySize, ArrayGet, ArrayPush, ArrayUpdate, SetSize, SetContains) |
| Porównania Int/Float | Operacje na łańcuchach |
| Logika boolowska | Wywołania funkcji |
| Dostęp/modyfikacja globalna | I/O, HTTP |
| Inkrementacja liczników | Operacje włókien |
| Operacje tablic: size, get, push, update | |
| Operacje zbiorów: size, contains | |
| RandomInt, RandomFloat, RandomChoice | |
| Pow, IntConcat, Has | |
| CastIntToFloat | |

---

## Компиляция трассы

Gdy trace wraca do `start_ip`, kompletny `Trace` jest przekazywany do `JIT::compile()` w Cranelift. Cranelift kompiluje sekwencję `TraceOp` do funkcji natywnej z sygnaturą `JITFunction` (3 parametry, zwraca `i32`).

Wskaźnik skompilowanej funkcji jest przechowywany w `Trace::native_ptr` (`AtomicPtr<u8>`), a `Arc<Trace>` jest wstawiany zarówno do `vm.traces` (globalnie współdzielony), jak i `trace_cache` (szybka ścieżka per-executor).

### Ustawienia kompilatora Cranelift

```rust
flag_builder.set("opt_level", "speed").unwrap();
flag_builder.set("use_colocated_libcalls", "false").unwrap();
flag_builder.set("is_pic", "false").unwrap();
flag_builder.set("regalloc_checker", "false").unwrap();
```

---

## Выполнение трассы

Przy każdej iteracji dyspozycji pętli, przed pobraniem następnego opcode:

```rust
if !self.is_recording && current_ip < self.trace_cache.len() {
    if let Some(trace) = &self.trace_cache[current_ip] {
        let jit_res = self.execute_trace(trace, ip, locals, glbs.as_mut().unwrap());
        if let Some(res) = jit_res { return res; }
        continue;
    }
}
```

Jeśli dostępna jest skompilowana funkcja natywna (`native_ptr != null`), jest wywoływana bezpośrednio przez `transmute` — całkowicie omijając interpreter dla całego ciała pętli. Jeśli JIT jeszcze tego nie skompilował, używana jest interpretowana ścieżka `TraceOp` jako krok pośredni.

---

## Method JIT — компиляция функций

Funkcje **bez pętli** (`has_loops = false`) i z mniej niż 500 instrukcjami bytecode kwalifikują się do kompilacji Method JIT.

### Wyzwalacz

Po **10 wywołaniach** tej samej funkcji kompilacja JIT jest uruchamiana asynchronicznie:

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

Skompilowany wskaźnik jest przechowywany w `FunctionChunk::jit_ptr` (`Arc<AtomicPtr<u8>>`), współdzielony między wszystkimi wątkami.

### Wykonanie szybkiej ścieżki

Przy każdym wywołaniu `run_frame_with_guard` wskaźnik JIT jest sprawdzany najpierw:

```rust
let jit_ptr = chunk.jit_ptr.load(Ordering::Relaxed);
if !jit_ptr.is_null() && !self.is_recording {
    let jit_fn: MethodJitFunction = unsafe { std::mem::transmute(jit_ptr) };
    // Prepare locals from params, call native function directly
    let res_bits = unsafe { jit_fn(locals.as_mut_ptr(), glbs_ptr, consts.as_ptr(), vm_ptr, executor_ptr) };
    return Some(Value(res_bits));
}
```

To omija całą pętlę interpretera, mechanizm śledzenia oraz alokację hot_counts/trace_cache dla skompilowanych funkcji.

### `compile_method` — pełna kompilacja bytecode

`JIT::compile_method` kompiluje kompletny `FunctionChunk` do kodu natywnego. Obsługuje większy podzbiór opcode niż kompilacja trace:

- Wszystkie opcode arytmetyczne, porównań i logiczne (tylko liczby całkowite)
- `LoadConst`, `Move`, `GetVar`, `SetVar` z pełnym liczeniem referencji
- Przepływ sterowania: `Jump`, `JumpIfFalse`, `JumpIfTrue`, `Return`, `ReturnVoid`
- Opcode pętli: `LoopNext`, `IncLocalLoopNext`, `IncVarLoopNext`, `ArrayLoopNext`
- `MethodCall` — dyspozycja przez zewnętrzną funkcję C `xcx_jit_method_dispatch`
- `Call` — wywołania rekurencyjne przez `xcx_jit_call_recursive` (inline obsługiwane są tylko wywołania samorekurencyjne; inne wywołania funkcji przechodzą do ścieżki interpretera)

Nieobsługiwane opcode powodują wczesny `return` z wartością zero (false), skutecznie wracając gracefully.

### Liczenie referencji w Method JIT

W przeciwieństwie do Tracing JIT (który pomija liczenie referencji dla szybkości), Method JIT zawiera pełne wywołania `inc_ref`/`dec_ref` przez zewnętrzne funkcje C `xcx_jit_inc_ref` i `xcx_jit_dec_ref`. Jest to wymagane, ponieważ skompilowane funkcje mogą trzymać i zwalniać wartości alokowane na stercie przez cały swój cykl życia.

Sprawdzenia wskaźników są wykonywane inline przez inspekcję bitów tagu przed wywołaniem helperów ref-count, unikając niepotrzebnego narzutu wywołań funkcji dla wartości skalarnych.

---

## Пул быстрых путей

Executor utrzymuje trzy pule, aby uniknąć alokacji O(N) w głęboko rekurencyjnych lub często wywoływanych funkcjach:

```rust
hot_counts_pool:   Vec<Vec<usize>>,
trace_cache_pool:  Vec<Vec<Option<Arc<Trace>>>>,
locals_pool:       Vec<Vec<Value>>,
```

Przy wejściu do `run_frame_with_guard` wektory są pobierane z puli (lub świeżo alokowane). Przy powrocie są zwracane do puli. Eliminuje to większość alokacji sterty per-wywołanie w gorących ścieżkach kodu.

`locals_pool` jest również używana przez `xcx_jit_call_recursive` dla szybkiej ścieżki samorekurencyjnych skompilowanych funkcji.

---

## Варианты TraceOp

### Ruch danych

| Wariant | Opis |
|---|-----
| `LoadConst { dst, val }` | Załaduj stałą do rejestru |
| `Move { dst, src }` | Skopiuj rejestr |
| `GetVar { dst, idx }` | Załaduj globalną |
| `SetVar { idx, src }` | Zapisz globalną |

### Arytmetyka całkowita

| Wariant | Opis |
|---|---|
| `AddInt / SubInt / MulInt` | Podstawowe operacje (wrapping) |
| `DivInt / ModInt` | Z `fail_ip` dla dzielenia przez zero / overflow |
| `PowInt` | Potęgowanie przez zewnętrzną funkcję C |
| `IntConcat` | Konkatenacja cyfr (123 ++ 456 = 123456) |

### Arytmetyka zmiennoprzecinkowa

| Wariant | Opis |
|---|---|
| `AddFloat / SubFloat / MulFloat` | Podstawowe operacje |
| `DivFloat / ModFloat` | Z `fail_ip` dla dzielenia przez zero |
| `PowFloat` | Potęgowanie przez zewnętrzną funkcję C |
| `CastIntToFloat` | konwersja int → float |

### Strażniki typów

| Wariant | Opis |
|---|---|
| `GuardInt { reg, ip }` | Side-exit do `ip`, jeśli rejestr nie jest Int |
| `GuardFloat { reg, ip }` | Side-exit do `ip`, jeśli rejestr nie jest Float |
| `GuardTrue { reg, fail_ip }` | Side-exit, jeśli wartość boolowska jest false |
| `GuardFalse { reg, fail_ip }` | Side-exit, jeśli wartość boolowska jest true |

### Porównania

| Wariant | Opis |
|---|---|
| `CmpInt { dst, src1, src2, cc }` | Porównanie całkowite z kodem warunku |
| `CmpFloat { dst, src1, src2, cc }` | Porównanie zmiennoprzecinkowe z kodem warunku |

**Kody warunków (cc):**

| cc | IntCC | FloatCC |
|---|---|---|
| 0 | Equal | Equal |
| 1 | NotEqual | NotEqual |
| 2 | SignedGreaterThan | GreaterThan |
| 3 | SignedLessThan | LessThan |
| 4 | SignedGreaterThanOrEqual | GreaterThanOrEqual |
| 5 | SignedLessThanOrEqual | LessThanOrEqual |

### Sterowanie pętlą

| Wariant | Opis |
|---|---|
| `LoopNextInt { reg, limit_reg, target, exit_ip }` | Inkrementacja + warunkowy skok dla pętli zakresowych |
| `IncVarLoopNext { g_idx, reg, limit_reg, target, exit_ip }` | Globalne liczniki + pętla zakresowa |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target, exit_ip }` | Lokalne liczniki + pętla zakresowa |
| `IncLocal { reg }` | Prosta inkrementacja zmiennej lokalnej |
| `IncVar { g_idx }` | Prosta inkrementacja zmiennej globalnej |
| `Jump { target_ip }` | Skok bezwarunkowy (wyzwala wykrywanie powrotu pętli) |

### Kolekcje

| Wariant | Opis |
|---|---|
| `ArraySize { dst, src }` | Pobierz rozmiar tablicy |
| `ArrayGet { dst, arr_reg, idx_reg, fail_ip }` | Pobierz element (z kontrolą granic) |
| `ArrayPush { arr_reg, val_reg }` | Dodaj element do tablicy |
| `ArrayUpdate { arr_reg, idx_reg, val_reg, fail_ip }` | Zaktualizuj element pod indeksem (z kontrolą granic) |
| `SetSize { dst, src }` | Pobierz rozmiar zbioru |
| `SetContains { dst, set_reg, val_reg }` | Sprawdź przynależność do zbioru |

### Losowość

| Wariant | Opis |
|---|---|
| `RandomInt { dst, min, max, step, has_step }` | Losowa liczba całkowita z zakresem/krokiem |
| `RandomFloat { dst, min, max, step, has_step, step_is_float }` | Losowa liczba zmiennoprzecinkowa z zakresem/krokiem |
| `RandomChoice { dst, src }` | Losowy element z kolekcji |

### Logika

`And / Or / Not` — operacje na bitach boolowskich

---

## Экспортируемые функции C

JIT wywołuje zewnętrzne funkcje Rust przez ABI C dla operacji, których nie można trywialnie inline'ować. Wszystkie są rejestrowane w `JITBuilder` podczas inicjalizacji JIT.

### Arytmetyka i matematyka

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

### Operacje na kolekcjach

```rust
#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_array_size(arr: Value) -> i64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_array_get(arr: Value, idx: i64) -> Value

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_array_push(arr: Value, val: Value)

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_array_update(arr: Value, idx: i64, val: Value) -> i32
// Returns 1 on success, 0 on out-of-bounds (triggers JIT side-exit)

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_set_size(set: Value) -> i64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_set_contains(set: Value, val: Value) -> bool
```

### Liczenie referencji

```rust
#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_inc_ref(v: Value)
// Increments Arc ref count if value is a pointer type

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_dec_ref(v: Value)
// Decrements Arc ref count if value is a pointer type; may free memory
```

Używane wyłącznie przez Method JIT do zarządzania cyklem życia wartości alokowanych na stercie. Tracing JIT pomija liczenie referencji dla maksymalnej wydajności pętli.

### Dyspozycja metod i wywołania rekurencyjne

```rust
#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_method_dispatch(
    dst: u8,
    kind: u8,           // MethodKind cast to u8
    receiver: Value,
    args_ptr: *const Value,
    arg_count: u8,
    locals_ptr: *mut Value,
    executor_ptr: *mut Executor,
)
// Dispatches a built-in method call from within a compiled function.
// Delegates to Executor::handle_method_call or handle_database_method.

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_call_recursive(
    func_id_idx: usize,
    params_ptr: *const Value,
    params_count: u8,
    vm_ptr: *const VM,
    executor_ptr: *mut Executor,
    globals_ptr: *mut Value,
) -> u64
// Calls a function from within a compiled function.
// Fast path: if the target function is also JIT-compiled, calls it directly
//            using locals from the executor's pool (avoids locking).
// Slow path: falls back to run_frame_with_guard for uncompiled functions.
```

---

## Конфигурация JIT

```rust
pub struct JIT {
    builder_context: FunctionBuilderContext,
    pub ctx:         codegen::Context,
    module:          JITModule,
}
```

Inicjalizacja:
1. Tworzy `JITBuilder` z natywnym ISA (auto-wykrywanie hosta przez `cranelift_native`)
2. Rejestruje wszystkie zewnętrzne symbole C (arytmetyka, kolekcje, liczenie referencji, dyspozycja)
3. Ustawia `opt_level: "speed"` dla maksymalnej wydajności
4. Tworzy `JITModule` zarządzający pamięcią wykonywalną

### Компиляция трассы (`compile`)

1. Czyści kontekst (`module.clear_context`)
2. Deklaruje funkcję z sygnaturą `JITFunction`: `(locals, globals, consts) -> i32`
3. Wykrywa obecność pętli (`has_loop`) — jeśli obecny jest `LoopNextInt`, `IncVarLoopNext` itd., owija ops w blok pętli Cranelift
4. Buduje IR Cranelift dla każdego `TraceOp`
5. Kompiluje i finalizuje definicje
6. Zwraca wskaźnik kodu (`get_finalized_function`)

### Kompilacja metody (`compile_method`)

1. Czyści kontekst (`module.clear_context`)
2. Deklaruje funkcję z sygnaturą `MethodJitFunction`: `(locals, globals, consts, vm, executor) -> u64`
3. Wstępnie skanuje bytecode pod kątem wszystkich celów skoków, aby utworzyć właściwą liczbę bloków Cranelift
4. Buduje IR Cranelift dla każdego obsługiwanego `OpCode`
5. Seal'uje wszystkie bloki i finalizuje
6. Zwraca wskaźnik kodu; przechowywany w `FunctionChunk::jit_ptr`

---

## Макросы NaN-boxing в JIT

JIT używa makr do pracy z wartościami NaN-boxed w IR Cranelift:

```rust
// Unpack integer (sign-extend from 48 bits)
macro_rules! unpack_int {
    ($val:expr) => {{
        let shl = b.ins().ishl_imm($val, 16);
        b.ins().sshr_imm(shl, 16)
    }};
}

// Pack integer
macro_rules! pack_int {
    ($raw:expr) => {{
        let lo = b.ins().band($raw, mask_48);
        b.ins().bor(qnan_tag_int, lo)
    }};
}

// Optimization: For NaN-boxed Int, increment is a simple iadd_imm(val, 1)
// on the entire 64-bit word — tag bits remain unaffected!
let lnxt_bits = b.ins().iadd_imm(lv, 1);
b.ins().store(trusted(), lnxt_bits, la, 0);
```

Ta optymalizacja działa, ponieważ tagi NaN są w wysokich bitach, a wartość Integer zajmuje dolne 48 bitów — dodanie 1 do całego słowa zmienia tylko część wartości (o ile nie ma overflow w domenie 48-bitowej). Ta sama szybka ścieżka inkrementacji jest używana dla zmiennych lokalnych i globalnych we wszystkich scalonych opcode pętli.
