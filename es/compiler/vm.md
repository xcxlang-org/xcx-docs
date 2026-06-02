> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/compiler/vm.md).

# Máquina Virtual de XCX (VM) — v3.1

La VM de XCX es un tiempo de ejecución personalizado **basado en registros** para ejecutar bytecode de XCX, aumentado por un compilador JIT de rastreo construido en Cranelift.

## Arquitectura

- **Archivo**: `src/backend/vm.rs`
- **Modelo de Ejecución**: Bucle Fetch-Decode-Execute (`execute_bytecode`) sobre un archivo de registros plano
- **Archivo de Registros**: Un `Vec<Value>` propio por frame, indexado por números de slot `u8`
- **Globales**: Un único `Vec<Value>` plano detrás de `Arc<RwLock<Vec<Value>>>`, compartido entre todos los hilos trabajadores
- **JIT**: `src/backend/jit.rs` — compilador de código nativo basado en Cranelift para trazas calientes

### Estado de la VM

```rust
pub struct VM {
    pub globals:     Arc<RwLock<Vec<Value>>>,
    pub error_count: AtomicUsize,
    pub traces:      Arc<RwLock<HashMap<usize, Arc<Trace>>>>,
    pub jit:         Mutex<JIT>,
}
```

`VM` está envuelta en `Arc<VM>` y compartida entre los hilos trabajadores HTTP. Cada trabajador crea su propio `Executor` con variables locales privadas.

### Estado del Executor

```rust
struct Executor {
    vm:               Arc<VM>,
    ctx:              SharedContext,
    current_spans:    Option<Arc<Vec<Span>>>,
    fiber_yielded:    bool,
    hot_counts:       Vec<usize>,         // contador de saltos hacia atrás por IP
    recording_trace:  Option<Trace>,      // traza siendo registrada
    is_recording:     bool,
    trace_cache:      Vec<Option<Arc<Trace>>>, // trazas compiladas indexadas por IP de inicio
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

`SharedContext` se clona de forma económica (dos incrementos de puntero `Arc`) y se pasa a cada hilo trabajador de forma independiente. No ocurre ninguna copia profunda.

---

## Representación de Valores: NaN-Boxing

Cada valor es un único `Value(u64)` — una palabra de 64 bits. XCX usa **NaN-boxing**: el patrón de bits de NaN silencioso de IEEE 754 se reutiliza como prefijo de etiqueta de tipo.

```
Disposición de bits: [63..52: exponente/QNAN] [51..48: etiqueta de tipo] [47..0: payload]

Float : almacenado directamente como bits f64 — NO tiene prefijo QNAN_BASE
Int   : QNAN_BASE | TAG_INT  | (valor i48 & 0x0000_FFFF_FFFF_FFFF)
Bool  : QNAN_BASE | TAG_BOOL | (0 o 1)
Date  : QNAN_BASE | TAG_DATE | (timestamp i48 ms)
Ptr   : QNAN_BASE | TAG_XXX  | (puntero & 0x0000_FFFF_FFFF_FFFF)
```

### Constantes de Etiqueta

| Constante | Valor | Tipo |
|---|---|---|
| `QNAN_BASE` | `0x7FF0_0000_0000_0000` | marcador NaN base |
| `TAG_INT`   | `0x0001_0000_0000_0000` | entero con signo de 48 bits |
| `TAG_BOOL`  | `0x0002_0000_0000_0000` | booleano (payload 0/1) |
| `TAG_DATE`  | `0x0003_0000_0000_0000` | timestamp de 48 bits (ms) |
| `TAG_STR`   | `0x0004_0000_0000_0000` | puntero `Arc<String>` |
| `TAG_ARR`   | `0x0005_0000_0000_0000` | puntero `Arc<RwLock<Vec<Value>>>` |
| `TAG_SET`   | `0x0006_0000_0000_0000` | puntero `Arc<RwLock<SetData>>` |
| `TAG_MAP`   | `0x0007_0000_0000_0000` | puntero `Arc<RwLock<Vec<(Value,Value)>>>` |
| `TAG_TBL`   | `0x0008_0000_0000_0000` | puntero `Arc<RwLock<TableData>>` |
| `TAG_FUNC`  | `0x0009_0000_0000_0000` | índice de función (u32) |
| `TAG_ROW`   | `0x000A_0000_0000_0000` | puntero `Arc<RowRef>` |
| `TAG_JSON`  | `0x000B_0000_0000_0000` | puntero `Arc<RwLock<serde_json::Value>>` |
| `TAG_FIB`   | `0x000C_0000_0000_0000` | puntero `Arc<RwLock<FiberState>>` |

Los payloads de puntero usan solo los 48 bits bajos — válido en todas las plataformas x86-64 y AArch64 donde los punteros de espacio de usuario caben en 48 bits.

### Conteo de Referencias para Valores Puntero

Los valores etiquetados como puntero llevan conteos de referencia `Arc`. La VM los gestiona manualmente mediante `inc_ref()` / `dec_ref()` en cada asignación, retorno y modificación de colección — asegurando que los objetos asignados en el heap (cadenas, arrays, JSON, fibers, etc.) sean liberados cuando ya no se referencian, sin un recolector de basura.

---

## Conjunto de Instrucciones (OpCodes)

Todos los opcodes son basados en registros: referencian slots de registro nombrados `u8` en lugar de una pila de operandos.

### Movimiento de Registros / Variables

| OpCode | Descripción |
|---|---|
| `LoadConst { dst, idx }` | Carga `constants[idx]` en el registro `dst` |
| `Move { dst, src }` | Copia el registro `src` en `dst` |
| `GetVar { dst, idx }` | Carga `globals[idx]` en `dst` (bloquea lectura de globales) |
| `SetVar { idx, src }` | Escribe `src` en `globals[idx]` (bloquea escritura de globales) |

### Aritmética

Todas las operaciones aritméticas son de 3 registros: `dst = src1 OP src2`. El despacho de tipos en tiempo de ejecución selecciona rutas de entero, float, concatenación de cadenas, aritmética de fechas u operaciones de conjuntos.

`Add`, `Sub`, `Mul`, `Div`, `Mod`, `Pow`, `IntConcat` (`++`)

### Comparación (resultado es Bool)

`Equal`, `NotEqual`, `Greater`, `Less`, `GreaterEqual`, `LessEqual`

### Lógica

`And { dst, src1, src2 }`, `Or { dst, src1, src2 }`, `Not { dst, src }`, `Has { dst, src1, src2 }`

### Flujo de Control

| OpCode | Descripción |
|---|---|
| `Jump { target }` | Salto incondicional; incrementa `hot_counts[target]` en saltos hacia atrás |
| `JumpIfFalse { src, target }` | Salta si `src` es `Bool(false)` |
| `JumpIfTrue { src, target }` | Salta si `src` es `Bool(true)` |
| `Call { dst, func_idx, base, arg_count }` | Llama función; args son `locals[base..base+arg_count]`; resultado → `dst` |
| `Return { src }` | Retorna el valor en `src` del frame actual |
| `ReturnVoid` | Retorna sin valor |
| `Halt` | Detiene la ejecución (explícito o en error irrecuperable) |

### Colecciones

| OpCode | Descripción |
|---|---|
| `ArrayInit { dst, base, count }` | Recopila `count` registros desde `base` → nuevo Array en `dst` |
| `SetInit { dst, base, count }` | Recopila `count` registros → nuevo Set en `dst` |
| `SetRange { dst, start, end, step, has_step }` | Construye Set por rango a partir de valores de registro |
| `MapInit { dst, base, count }` | Recopila `count` pares clave-valor (registros en pares alternos desde `base`) → nuevo Map |
| `TableInit { dst, skeleton_idx, base, row_count }` | Construye Table desde constante de esquema de columnas + valores de fila |

### Operaciones de Conjunto

`SetUnion`, `SetIntersection`, `SetDifference`, `SetSymDifference` — todos de 3 registros, ambos operandos deben ser `TAG_SET`.

### Despacho de Métodos

| OpCode | Descripción |
|---|---|
| `MethodCall { dst, kind, base, arg_count }` | Despacha método integrado por enum `MethodKind` — sin búsqueda de cadenas en tiempo de ejecución |
| `MethodCallCustom { dst, method_name_idx, base, arg_count }` | Despacha método dinámico (campo JSON, alias) por cadena de la tabla de constantes |

`base` apunta al registro receptor; los argumentos son `locals[base+1..base+1+arg_count]`. `MethodKind` es un enum `#[derive(Copy)]` que cubre ~50 métodos integrados (`Push`, `Pop`, `Get`, `Insert`, `Update`, `Delete`, `Where`, `Join`, `Sort`, `Format`, `Next`, `IsDone`, `Close`, etc.). El compilador resuelve nombres de métodos a variantes de `MethodKind` en tiempo de compilación mediante `map_method_kind()`.

### Operaciones de Fiber

| OpCode | Descripción |
|---|---|
| `FiberCreate { dst, func_idx, base, arg_count }` | Asigna `FiberState`, pre-rellena locals desde args, almacena `Fiber` en `dst` |
| `Yield { src }` | Suspende el fiber, retorna el valor en `src` al llamador |
| `YieldVoid` | Suspende fiber void |

### E/S y Sistema

| OpCode | Descripción |
|---|---|
| `Print { src }` | Imprime `locals[src]` en stdout |
| `Input { dst }` | Lee línea desde stdin → `dst` |
| `Wait { src }` | Duerme por `src` milisegundos |
| `HaltAlert { src }` | Imprime mensaje de alerta, continúa ejecución |
| `HaltError { src }` | Imprime error + info de span, detiene frame, incrementa conteo de errores |
| `HaltFatal { src }` | Imprime fatal + info de span, detiene frame, incrementa conteo de errores |
| `TerminalExit` | `std::process::exit(0)` |
| `TerminalClear` | Limpia terminal mediante secuencia de escape ANSI o comando del SO |
| `TerminalRun { dst, cmd_src }` | Ejecuta comando externo, resultado → `dst` |
| `EnvGet { dst, src }` | Lee variable de entorno nombrada por `src` → `dst` |
| `EnvArgs { dst }` | Inserta `Array<String>` de argumentos CLI → `dst` |

### HTTP

| OpCode | Descripción |
|---|---|
| `HttpCall { dst, method_idx, url_src, body_src }` | Llamada HTTP simple (GET/POST/etc.) mediante `ureq`, resultado JSON → `dst` |
| `HttpRequest { dst, arg_src }` | Llamada HTTP completa desde mapa de configuración (method, url, headers, body, timeout) → `dst` |
| `HttpRespond { status_src, body_src, headers_src }` | Envía respuesta HTTP desde dentro de un fiber manejador; activa `Yield` para devolver el control |
| `HttpServe { func_idx, port_src, host_src, workers_src, routes_src }` | Inicia servidor `tiny_http`, lanza hilos trabajadores, bloquea hilo principal hasta `SHUTDOWN` |

### Almacenamiento

`StoreWrite { base }`, `StoreRead { dst, base }`, `StoreAppend { base }`, `StoreExists { dst, base }`, `StoreDelete { base }`

### JSON

`JsonParse { dst, src }`, `JsonBind { idx, json_src, path_src }`, `JsonBindLocal { dst, json_src, path_src }`, `JsonInject { table_idx, json_src, mapping_src }`, `JsonInjectLocal { table_reg, json_src, mapping_src }`

### Castings de Tipo

`CastInt { dst, src }`, `CastFloat { dst, src }`, `CastString { dst, src }`, `CastBool { dst, src }`

### Criptografía y Fechas

`CryptoHash { dst, pass_src, alg_src }`, `CryptoVerify { dst, pass_src, hash_src, alg_src }`, `CryptoToken { dst, len_src }`, `DateNow { dst }`

### Optimizaciones de Bucle

Estos opcodes son emitidos por el compilador para fusionar patrones comunes de contador de bucle en instrucciones únicas, reduciendo la sobrecarga de despacho y mejorando la trazabilidad del JIT.

| OpCode | Descripción |
|---|---|
| `IncLocal { reg }` | Incrementa el entero en el registro `reg` en 1 |
| `IncVar { idx }` | Incrementa el global en `idx` en 1 |
| `LoopNext { reg, limit_reg, target }` | Incrementa `reg`, salta a `target` si `reg <= limit_reg`, de lo contrario continúa |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target }` | Incrementa `inc_reg` (contador separado, p.ej. índice de array), incrementa `reg` (var de bucle), salto condicional |
| `IncVarLoopNext { g_idx, reg, limit_reg, target }` | Como `IncLocalLoopNext` pero `g_idx` es un contador global |

---

## Compilador JIT de Rastreo

### Descripción General

XCX 2.2 incluye un **JIT de rastreo** que compila automáticamente bucles calientes a código máquina nativo usando el framework de generación de código [Cranelift](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift).

El JIT es completamente transparente para el programador — se activa automáticamente y recurre al intérprete en guardas de tipo u operaciones no soportadas.

### Detección de Trazas

En cada `Jump { target }` donde `target < current_ip` (un salto hacia atrás, es decir, arco de retorno de bucle):

1. `hot_counts[target]` se incrementa.
2. Cuando `hot_counts[target] >= 50`, se inicia una nueva `Trace` con `start_ip = target`.
3. El registro de trazas está bloqueado: solo comienza si `trace_cache[target].is_none()` (no existe traza compilada aún) y `is_recording` es falso.

### Registro de Trazas

Mientras `is_recording` es verdadero, el intérprete ejecuta cada opcode normalmente **y** registra un `TraceOp` especializado para los tipos en tiempo de ejecución actuales. Por ejemplo:

- Un `Add` sobre dos registros enteros registra `GuardInt { reg: src1 }`, `GuardInt { reg: src2 }`, `AddInt { dst, src1, src2 }` — no un `Add` genérico.
- Un `JumpIfFalse` que **no** se toma registra `GuardTrue { reg: src, fail_ip: target }` — afirmando que la rama es siempre falsa.
- Un `JumpIfFalse` que **se toma** registra `GuardFalse { reg: src, fail_ip: next_ip }`.

Si un opcode no puede ser trazado (llamadas a métodos complejas, operaciones de cadenas, etc.), el registro se aborta y `is_recording` vuelve a false.

### Compilación de Trazas

Cuando el bucle trazado ejecuta de vuelta a `start_ip`, la `Trace` completa se entrega a la función `JIT::compile()` de Cranelift (`src/backend/jit.rs`). Cranelift compila la secuencia de `TraceOp` a una función nativa con la firma:

```rust
unsafe extern "C" fn(
    locals_ptr: *mut Value,
    globals_ptr: *mut Value,
    consts_ptr:  *const Value,
) -> i32
```

El valor de retorno es el siguiente IP en el que reanudar (0 = continuar normalmente, positivo = IP de salida lateral, negativo = detener).

El puntero a la función compilada se almacena en `Trace::native_ptr` (un `AtomicPtr<u8>`) y el `Arc<Trace>` se inserta tanto en `vm.traces` (compartido globalmente) como en `trace_cache` (ruta rápida por executor).

### Ejecución de Trazas

En cada iteración del bucle de despacho, antes de obtener el siguiente opcode:

```
if trace_cache[current_ip].is_some() {
    execute_trace(trace, ip, locals, &mut glbs)
    continue
}
```

Si hay una función nativa compilada disponible (`native_ptr != null`), se llama mediante `transmute` directamente — omitiendo completamente el intérprete para todo el cuerpo del bucle. Si el JIT aún no ha compilado, se usa la ruta interpretada de `TraceOp` como paso intermedio.

### Variantes de TraceOp

| Variante | Descripción |
|---|---|
| `LoadConst`, `Move` | Movimientos de registro con valores constantes |
| `AddInt/SubInt/MulInt/DivInt/ModInt` | Aritmética entera con IP de fallo para div/mod por cero |
| `AddFloat/SubFloat/MulFloat/DivFloat/ModFloat` | Aritmética de float |
| `CmpInt / CmpFloat` | Comparación usando código de condición `cc: u8` |
| `GuardInt / GuardFloat` | Guarda de tipo — sale de la traza si el registro tiene tipo incorrecto |
| `GuardTrue / GuardFalse` | Guarda de rama — sale de la traza en dirección de rama inesperada |
| `CastIntToFloat` | Amplía registro int a float |
| `IncLocal / IncVar` | Incremento de registro único/global |
| `LoopNextInt` | Incremento combinado + salto condicional para bucles de rango |
| `IncVarLoopNext / IncLocalLoopNext` | Variantes fusionadas para bucles de array y for-range |
| `GetVar / SetVar` | Acceso a variables globales |
| `And / Or / Not` | Lógica booleana |
| `Jump` | Salto incondicional (activa detección de retorno de bucle en traza) |

---

## Flujo de Ejecución

```
VM::run(main_chunk, ctx)                        [Arc<VM>]
  └─ Executor::run_frame_owned(main_chunk)
       └─ execute_bytecode(bytecode, &mut ip, &mut locals)
            │
            ├─ [Ruta rápida JIT] si trace_cache[ip].is_some():
            │     execute_trace(trace, ip, locals, globals)
            │     → retorna siguiente IP o None
            │
            └─ [Ruta intérprete] obtiene opcode, ejecuta
                 ├─ Continue      → avanza ip normalmente
                 ├─ Jump(t)       → ip = t; incrementa hot_counts si es hacia atrás
                 ├─ Return(val)   → sale del frame, retorna val
                 ├─ Yield(val)    → suspende (fiber), retorna val al llamador
                 └─ Halt          → detiene, incrementa error_count
```

Las funciones se llaman mediante `run_frame(func_id, params)`, que crea un vector `locals` nuevo pre-dimensionado a `chunk.max_locals`. `current_spans` se intercambia con la tabla de spans de la función llamada y se restaura al retornar.

---

## Modelo de Ejecución de Fibers

Los fibers son **corrutinas cooperativas**, no hilos.

```rust
pub struct FiberState {
    pub func_id:       usize,
    pub ip:            usize,
    pub locals:        Vec<Value>,    // se mueve durante la reanudación, se devuelve después
    pub is_done:       bool,
    pub yielded_value: Option<Value>, // valor en caché para el patrón IsDone + Next
}
```

### Secuencia de Reanudación (`resume_fiber`)

1. Se leen `func_id`, `ip` y se **mueven** `locals` fuera de `FiberState` mediante `std::mem::take` — sin clon.
2. Se ejecuta `execute_bytecode` desde `fiber.ip` con los locals movidos.
3. En `Yield`: se establece `fiber_yielded = true`. Los locals se mueven de vuelta a `FiberState`. Se actualiza `fiber.ip`. Se retorna el valor producido.
4. En `Return` / fin del bytecode: se establece `fiber.is_done = true`. Se retorna el valor final.

La reanudación/suspensión no implica asignación en el heap más allá de la creación inicial del `Vec` — solo movimientos.

### Patrón IsDone / Next

`IsDone` comprueba `FiberState::is_done`, teniendo en cuenta si `yielded_value` está en caché. `Next` toma el `yielded_value` en caché si está presente (de una reanudación anterior que ya se ejecutó), o llama a `resume_fiber`. Esto asegura que un bucle `for x in fiber` nunca avance el fiber dos veces.

### Bucle For sobre Fiber (`ForIterType::Fiber`)

El compilador emite:
1. `MethodCall(IsDone)` → `JumpIfTrue` para salir
2. `MethodCall(Next)` → asigna a variable de bucle
3. Cuerpo del bucle
4. `Jump` de vuelta al paso 1
5. En `break`: `MethodCall(Close, base=fiber_reg)` marca el fiber como terminado antes de saltar fuera

---

## Servidor HTTP (`HttpServe`)

`HttpServe` inicia un `tiny_http::Server` y lanza N hilos del SO:

```rust
for _ in 0..workers {
    let server  = server.clone();    // Arc<tiny_http::Server>
    let vm      = vm_arc.clone();    // Arc<VM>
    let ctx     = self.ctx.clone();  // SharedContext (dos clones Arc)
    let routes  = routes.clone();    // Arc<Vec<(String, usize)>>
    std::thread::spawn(move || { /* recv → match route → run handler fiber */ });
}
```

Cada trabajador ejecuta su propio `Executor` con sus propios locals. Los globales se comparten mediante `Arc<RwLock<Vec<Value>>>`.

### Manejo de Peticiones

Para cada petición entrante, el trabajador:
1. Hace coincidir la clave `"METHOD /path"` con la tabla de rutas (sin distinción de mayúsculas/minúsculas).
2. Construye un objeto JSON `{ method, url, body, ip, headers }` como `Value::Json`.
3. Almacena la `tiny_http::Request` en `Arc<Mutex<Option<tiny_http::Request>>>` y la pasa a un `Executor` nuevo mediante `http_req`.
4. Ejecuta el fiber manejador coincidente de forma síncrona en el `Executor` de ese trabajador.
5. Cuando el manejador llama a `net.respond(...)`, la VM ejecuta `HttpRespond` que envía la respuesta y retorna `OpResult::Yield` para terminar el manejador.
6. Si el manejador sale sin llamar a `net.respond`, el trabajador envía una respuesta de fallback `500`.

### Apagado Controlado

`SHUTDOWN` es un `pub static AtomicBool` en `vm.rs`. Un manejador de Ctrl+C en `main.rs` lo establece en `true`. Los trabajadores lo comprueban cada ciclo `recv_timeout(100ms)`. El hilo principal se bloquea en un bucle `sleep(500ms)` también sondeando `SHUTDOWN`. Una vez establecido, todos los bucles salen y el proceso termina limpiamente.

---

## Compilador (`src/backend/mod.rs`)

### Asignación de Registros

`FunctionCompiler` rastrea el siguiente registro disponible con `next_local: usize`:

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

Los locals (variables con nombre) reciben un slot mediante `define_local(id, slot)` y se almacenan en `scopes: Vec<HashMap<StringId, usize>>`. Los temporales usan `push_reg()`/`pop_reg()` — se reutilizan una vez que el resultado de la expresión es consumido.

`max_locals_used` se registra para que `FunctionChunk::max_locals` pueda pre-asignar un vector de locals del tamaño exacto correcto cuando se llama la función.

### Deduplicación de Constantes

`CompileContext::add_constant` deduplica constantes de cadena mediante `string_constants: HashMap<String, usize>`. Las constantes de cadena duplicadas (p.ej., `"insert"` apareciendo muchas veces como argumento de nombre de método) reutilizan el mismo slot de la tabla de constantes.

### Compilación en Dos Pasadas

**Pasada 1** — `register_globals_recursive`:
- Asigna un índice de slot a cada variable global e instancia de fiber-decl
- Asigna un índice de función a cada función/fiber
- Pre-asigna slots `FunctionChunk` vacíos en `functions: Vec<FunctionChunk>`

**Pasada 2** — `compile_stmt` / `compile_expr`:
- Emite bytecode emparejado con spans mediante `emit(op, span)`
- Las sentencias de nivel superior en `main` usan `GetVar`/`SetVar` (globales); las sentencias anidadas usan registros

### `FunctionChunk`

```rust
pub struct FunctionChunk {
    pub bytecode:   Arc<Vec<OpCode>>,
    pub spans:      Arc<Vec<Span>>,    // spans[i] corresponde a bytecode[i]
    pub is_fiber:   bool,
    pub max_locals: usize,
}
```

El bytecode y los spans están envueltos en `Arc` para poder compartirlos entre hilos trabajadores HTTP sin copiarlos.

---

## Reporte de Errores en Tiempo de Ejecución

Cada error en tiempo de ejecución en la VM añade `self.current_span_info(ip)` que retorna `" [line: X, col: Y]"` buscando `current_spans[ip - 1]`. Ejemplo:

```
ERROR: R303: Array index out of bounds: 5 [line: 14, col: 7]
```

`current_spans` se intercambia con el `Arc<Vec<Span>>` correcto en cada llamada `run_frame` y se restaura al salir.

---

## Modelo de Memoria

- **Sin recolector de basura**. Conteo de referencias mediante `Arc` (con `inc_ref`/`dec_ref` manuales para valores puntero con NaN-boxing).
- **Valores escalares** (Int, Float, Bool, Date, índice de función): almacenados completamente en el `u64` — cero asignación en heap.
- **Valores de colección**: `Arc<RwLock<T>>` proporciona propiedad compartida. Clonar un `Value` de colección solo incrementa el contador `Arc`.
- **Mutaciones**: `.insert()`, `.update()`, `.delete()` adquieren un bloqueo de escritura. Todos los handles a la misma colección ven el cambio.
- **Métodos de solo lectura**: `.size()`, `.get()`, `.contains()` adquieren un bloqueo de lectura — se permiten múltiples lectores concurrentes.

---

## Controles de Seguridad

### Protección SSRF de Red (`is_safe_url`)

Objetivos bloqueados:
- URLs `file://`
- `169.254.x.x` (link-local / endpoint de metadatos AWS)
- Rangos privados: `10.x`, `192.168.x`, `172.16–31.x` (cuando no es localhost)

Se aplica tanto a `HttpCall` como a `HttpRequest`.

### Límite de Cuerpo HTTP

En `HttpServe`, el cuerpo de respuesta se comprueba tras `into_string()`. Si supera **10 MB**, se devuelve un JSON de error `413` en lugar del cuerpo real.

### Cabeceras CORS

Todas las respuestas de `HttpServe` incluyen automáticamente:
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

Las peticiones de preflight `OPTIONS` reciben una respuesta `204` sin invocar ningún manejador.