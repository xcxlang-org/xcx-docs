> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/compiler/backend.md).

# XCX Backend — Compilador y VM

> **Archivos:** `src/backend/mod.rs`, `src/backend/vm.rs`

---

## Tabla de Contenidos

1. [Compilador de Bytecode](#compilador-de-bytecode)
2. [Representación de Valores — NaN-Boxing](#representación-de-valores--nan-boxing)
3. [Conjunto de Instrucciones (OpCodes)](#conjunto-de-instrucciones-opcodes)
4. [Arquitectura de la VM](#arquitectura-de-la-vm)
5. [Modelo de Ejecución de Fibers](#modelo-de-ejecución-de-fibers)
6. [Servidor HTTP](#servidor-http)
7. [Gestión de Memoria](#gestión-de-memoria)
8. [Optimizaciones de Bucles](#optimizaciones-de-bucles)

---

## Compilador de Bytecode

**Archivo:** `src/backend/mod.rs`

### Asignación de Registros

`FunctionCompiler` rastrea el siguiente registro disponible a través de `next_local: usize`:

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

Las variables locales con nombre se asignan a un slot mediante `define_local(id, slot)` y se almacenan en `scopes: Vec<HashMap<StringId, usize>>`. Las variables temporales usan `push_reg()`/`pop_reg()` — se reutilizan cuando se consume el resultado de una expresión.

`max_locals_used` se registra para que `FunctionChunk::max_locals` pueda pre-asignar un vector de locales del tamaño exacto cuando se llama a la función.

### Deduplicación de Constantes

`CompileContext::add_constant` deduplica constantes de cadena mediante `string_constants: HashMap<String, usize>`. Las constantes de cadena duplicadas (por ejemplo, `"insert"` que aparece varias veces como argumento de nombre de método) reutilizan el mismo slot de la tabla de constantes.

### Compilación en Dos Pasadas

**Pasada 1** — `register_globals_recursive`:
- Asigna un índice de slot a cada variable global e instancia de fiber-decl
- Asigna un índice de función a cada función/fiber
- Pre-asigna slots `FunctionChunk` vacíos en `functions: Vec<FunctionChunk>`

**Pasada 2** — `compile_stmt` / `compile_expr`:
- Emite bytecode emparejado con spans mediante `emit(op, span)`
- Las sentencias de nivel superior en `main` usan `GetVar`/`SetVar` (globales); las sentencias anidadas usan registros

### FunctionChunk

```rust
pub struct FunctionChunk {
    pub bytecode:   Arc<Vec<OpCode>>,
    pub spans:      Arc<Vec<Span>>,    // spans[i] corresponds to bytecode[i]
    pub is_fiber:   bool,
    pub max_locals: usize,
}
```

El bytecode y los spans se envuelven en `Arc` para poder compartirlos entre hilos trabajadores HTTP sin copia.

---

## Representación de Valores — NaN-Boxing

Cada valor es un único `Value(u64)` — una palabra de 64 bits. XCX usa **NaN-boxing**: el patrón de bits de NaN silencioso de IEEE 754 se reutiliza como prefijo de etiqueta de tipo.

```
Disposición de bits: [63..52: exponent/QNAN] [51..48: type tag] [47..0: payload]

Float : almacenado directamente como bits f64 — NO tiene el prefijo QNAN_BASE establecido
Int   : QNAN_BASE | TAG_INT  | (valor i48 & 0x0000_FFFF_FFFF_FFFF)
Bool  : QNAN_BASE | TAG_BOOL | (0 o 1)
Date  : QNAN_BASE | TAG_DATE | (timestamp i48 ms)
Ptr   : QNAN_BASE | TAG_XXX  | (puntero & 0x0000_FFFF_FFFF_FFFF)
```

### Constantes de Etiqueta

| Constante | Valor | Tipo |
|---|---|---|
| `QNAN_BASE` | `0x7FF0_0000_0000_0000` | marcador NaN base |
| `TAG_INT` | `0x0001_0000_0000_0000` | entero de 48 bits con signo |
| `TAG_BOOL` | `0x0002_0000_0000_0000` | booleano (payload 0/1) |
| `TAG_DATE` | `0x0003_0000_0000_0000` | timestamp de 48 bits (ms) |
| `TAG_STR` | `0x0004_0000_0000_0000` | puntero `Arc<Vec<u8>>` |
| `TAG_ARR` | `0x0005_0000_0000_0000` | puntero `Arc<RwLock<Vec<Value>>>` |
| `TAG_SET` | `0x0006_0000_0000_0000` | puntero `Arc<RwLock<SetData>>` |
| `TAG_MAP` | `0x0007_0000_0000_0000` | puntero `Arc<RwLock<Vec<(Value,Value)>>>` |
| `TAG_TBL` | `0x0008_0000_0000_0000` | puntero `Arc<RwLock<TableData>>` |
| `TAG_FUNC` | `0x0009_0000_0000_0000` | índice de función (u32) |
| `TAG_ROW` | `0x000A_0000_0000_0000` | puntero `Arc<RowRef>` |
| `TAG_JSON` | `0x000B_0000_0000_0000` | puntero `Arc<RwLock<serde_json::Value>>` |
| `TAG_FIB` | `0x000C_0000_0000_0000` | puntero `Arc<RwLock<FiberState>>` |
| `TAG_DB` | `0x000D_0000_0000_0000` | puntero `Arc<DatabaseData>` |

Los payloads de puntero usan solo los 48 bits bajos — válido en todas las plataformas x86-64 y AArch64 donde los punteros de espacio de usuario caben en 48 bits.

### Conteo de Referencias

Los valores etiquetados como puntero llevan conteos de referencia `Arc`. La VM los gestiona manualmente mediante `inc_ref()` / `dec_ref()` en cada asignación, retorno y modificación de colección — asegurando que los objetos en heap se liberen cuando ya no se referencian, sin recolector de basura.

---

## Conjunto de Instrucciones (OpCodes)

Todos los opcodes son basados en registros: referencian slots de registro `u8` con nombre en lugar de una pila de operandos.

### Movimiento de Registro / Variable

| OpCode | Descripción |
|---|---|
| `LoadConst { dst, idx }` | Carga `constants[idx]` en el registro `dst` |
| `Move { dst, src }` | Copia el registro `src` en `dst` |
| `GetVar { dst, idx }` | Carga `globals[idx]` en `dst` (bloqueo de lectura de globales) |
| `SetVar { idx, src }` | Escribe `src` en `globals[idx]` (bloqueo de escritura de globales) |

### Aritmética

Todas las operaciones aritméticas son de 3 registros: `dst = src1 OP src2`. El despacho de tipos en tiempo de ejecución selecciona rutas de entero, float, concatenación de cadenas, aritmética de fechas u operaciones de conjuntos.

`Add`, `Sub`, `Mul`, `Div`, `Mod`, `Pow`, `IntConcat` (`++`)

### Comparaciones (resultado Bool)

`Equal`, `NotEqual`, `Greater`, `Less`, `GreaterEqual`, `LessEqual`

### Lógica

`And { dst, src1, src2 }`, `Or { dst, src1, src2 }`, `Not { dst, src }`, `Has { dst, src1, src2 }`

### Flujo de Control

| OpCode | Descripción |
|---|---|
| `Jump { target }` | Salto incondicional; incrementa `hot_counts[target]` en saltos hacia atrás |
| `JumpIfFalse { src, target }` | Salta si `src` es `Bool(false)` |
| `JumpIfTrue { src, target }` | Salta si `src` es `Bool(true)` |
| `Call { dst, func_idx, base, arg_count }` | Llama función; resultado → `dst` |
| `Return { src }` | Retorna el valor en `src` del frame actual |
| `ReturnVoid` | Retorna sin valor |
| `Halt` | Detiene la ejecución |

### Colecciones

| OpCode | Descripción |
|---|---|
| `ArrayInit { dst, base, count }` | Recopila `count` registros desde `base` → nuevo array en `dst` |
| `SetInit { dst, base, count }` | Recopila `count` registros → nuevo conjunto en `dst` |
| `SetRange { dst, start, end, step, has_step }` | Construye conjunto por rango a partir de valores de registro |
| `MapInit { dst, base, count }` | Recopila `count` pares clave-valor → nuevo mapa en `dst` |
| `TableInit { dst, skeleton_idx, base, row_count }` | Construye tabla desde constante de esquema de columnas + valores de fila |

### Operaciones de Conjunto

`SetUnion`, `SetIntersection`, `SetDifference`, `SetSymDifference` — todos de 3 registros, ambos operandos deben ser `TAG_SET`.

### Despacho de Métodos

| OpCode | Descripción |
|---|---|
| `MethodCall { dst, kind, base, arg_count }` | Despacha método integrado por enum `MethodKind` — sin búsqueda de cadenas en tiempo de ejecución |
| `MethodCallCustom { dst, method_name_idx, base, arg_count }` | Despacha método dinámico (campo JSON, alias) por cadena de la tabla de constantes |
| `MethodCallNamed { dst, kind, base, arg_count, names_idx }` | Llamada de método con argumentos nombrados |

`base` apunta al registro receptor; los argumentos son `locals[base+1..base+1+arg_count]`. `MethodKind` es un enum `#[derive(Copy)]` que cubre ~50 métodos integrados (`Push`, `Pop`, `Get`, `Insert`, `Update`, `Delete`, `Where`, `Join`, `Sort`, `Format`, `Next`, `IsDone`, `Close`, etc.).

### Operaciones de Fiber

| OpCode | Descripción |
|---|---|
| `FiberCreate { dst, func_idx, base, arg_count }` | Asigna `FiberState`, pre-rellena locales desde args, almacena `Fiber` en `dst` |
| `Yield { src }` | Suspende el fiber, retorna el valor en `src` al llamador |
| `YieldVoid` | Suspende fiber void |

### E/S y Sistema

| OpCode | Descripción |
|---|---|
| `Print { src }` | Imprime `locals[src]` en stdout |
| `Input { dst, ty }` | Lee línea desde stdin → `dst` con conversión de tipo |
| `Wait { src }` | Duerme por `src` milisegundos |
| `HaltAlert { src }` | Imprime mensaje de alerta, continúa ejecución |
| `HaltError { src }` | Imprime error + info de span, detiene frame, incrementa error_count |
| `HaltFatal { src }` | Imprime fatal + info de span, detiene frame, incrementa error_count |
| `TerminalExit` | `std::process::exit(0)` |
| `TerminalClear` | Limpia terminal mediante secuencia de escape ANSI |
| `TerminalRaw / TerminalNormal` | Activa/desactiva modo terminal raw |
| `TerminalCursor { on }` | Muestra/oculta cursor |
| `TerminalMove { x_src, y_src }` | Mueve cursor |
| `TerminalWrite { src }` | Escribe sin salto de línea |
| `InputKey { dst }` | Lee tecla (no bloqueante) |
| `InputKeyWait { dst }` | Lee tecla (bloqueante) |
| `InputReady { dst }` | Comprueba si hay entrada disponible |
| `EnvGet { dst, src }` | Lee variable de entorno |
| `EnvArgs { dst }` | Obtiene argumentos CLI |

### HTTP

| OpCode | Descripción |
|---|---|
| `HttpCall { dst, method_idx, url_src, body_src }` | Llamada HTTP simple mediante `ureq`, resultado JSON → `dst` |
| `HttpRequest { dst, arg_src }` | Llamada HTTP completa desde mapa de configuración (método, url, headers, body, timeout) → `dst` |
| `HttpRespond { status_src, body_src, headers_src }` | Envía respuesta HTTP desde dentro de un fiber manejador |
| `HttpServe { func_idx, port_src, host_src, workers_src, routes_src }` | Inicia servidor `tiny_http`, lanza hilos trabajadores |

### Almacenamiento

`StoreWrite`, `StoreRead`, `StoreAppend`, `StoreExists`, `StoreDelete`, `StoreList`, `StoreIsDir`, `StoreSize`, `StoreMkdir`, `StoreGlob`, `StoreZip`, `StoreUnzip`

### JSON

`JsonParse { dst, src }`, `JsonBind { idx, json_src, path_src }`, `JsonBindLocal { dst, json_src, path_src }`, `JsonInject { table_idx, json_src, mapping_src }`, `JsonInjectLocal { table_reg, json_src, mapping_src }`

### Castings de Tipo

`CastInt { dst, src }`, `CastFloat { dst, src }`, `CastString { dst, src }`, `CastBool { dst, src }`

### Criptografía y Fechas

`CryptoHash`, `CryptoVerify`, `CryptoToken`, `DateNow`

### Base de Datos

`DatabaseInit { dst, engine_src, path_src, tables_base_reg, table_count }`

---

## Arquitectura de la VM

**Archivo:** `src/backend/vm.rs`

### Estado de la VM

```rust
pub struct VM {
    pub globals:     Arc<RwLock<Vec<Value>>>,
    pub error_count: AtomicUsize,
    pub traces:      Arc<RwLock<HashMap<usize, Arc<Trace>>>>,
    pub jit:         Mutex<JIT>,
}
```

`VM` está envuelta en `Arc<VM>` y compartida entre hilos trabajadores HTTP. Cada trabajador crea su propio `Executor` con locales privados.

### Estado del Executor

```rust
struct Executor {
    vm:               Arc<VM>,
    ctx:              SharedContext,
    current_spans:    Option<Arc<Vec<Span>>>,
    fiber_yielded:    bool,
    hot_counts:       Vec<usize>,
    recording_trace:  Option<Trace>,
    is_recording:     bool,
    trace_cache:      Vec<Option<Arc<Trace>>>,
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

`SharedContext` se clona de forma económica (dos incrementos de puntero `Arc`) y se pasa a cada hilo trabajador de forma independiente. No ocurre ninguna copia profunda.

### Flujo de Ejecución

```
VM::run(main_chunk, ctx)                        [Arc<VM>]
  └─ Executor::run_frame_owned(main_chunk)
       └─ execute_bytecode_inner(bytecode, &mut ip, &mut locals, ...)
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

---

## Modelo de Ejecución de Fibers

Los fibers son **corrutinas cooperativas**, no hilos del SO.

```rust
pub struct FiberState {
    pub func_id:       usize,
    pub ip:            usize,
    pub locals:        Vec<Value>,    // movidos durante la reanudación, devueltos después
    pub is_done:       bool,
    pub yielded_value: Option<Value>, // caché para el patrón IsDone + Next
}
```

### Secuencia de Reanudación (`resume_fiber`)

1. Se leen `func_id`, `ip` y se **mueven** `locals` fuera de `FiberState` mediante `std::mem::take` — sin clon.
2. Se ejecuta `execute_bytecode` desde `fiber.ip` con los locales movidos.
3. En `Yield`: se establece `fiber_yielded = true`. Los locales se mueven de vuelta a `FiberState`. Se actualiza `fiber.ip`. Se retorna el valor producido.
4. En `Return` / fin del bytecode: se establece `fiber.is_done = true`. Se retorna el valor final.

La reanudación/suspensión no implica asignaciones en heap más allá de la creación inicial del `Vec` — solo movimientos.

### Patrón IsDone / Next

`IsDone` comprueba `FiberState::is_done`, teniendo en cuenta si `yielded_value` está en caché. `Next` toma el `yielded_value` en caché si está presente, o llama a `resume_fiber`. Esto asegura que un bucle `for x in fiber` nunca avance el fiber dos veces.

### Bucle For sobre Fiber (`ForIterType::Fiber`)

El compilador emite:
1. `MethodCall(IsDone)` → `JumpIfTrue` para salir
2. `MethodCall(Next)` → asigna a variable de bucle
3. Cuerpo del bucle
4. `Jump` de vuelta al paso 1
5. En `break`: `MethodCall(Close, base=fiber_reg)` marca el fiber como terminado antes de saltar fuera

---

## Servidor HTTP

`HttpServe` inicia un servidor `tiny_http` y lanza N hilos del SO:

```rust
for _ in 0..workers {
    let server  = server.clone();    // Arc<tiny_http::Server>
    let vm      = vm_arc.clone();    // Arc<VM>
    let ctx     = self.ctx.clone();  // SharedContext (dos clones Arc)
    let routes  = routes.clone();    // Arc<Vec<(String, usize)>>
    std::thread::spawn(move || { /* recv → match route → run handler fiber */ });
}
```

Cada trabajador ejecuta su propio `Executor` con sus propios locales. Los globales se comparten mediante `Arc<RwLock<Vec<Value>>>`.

### Manejo de Peticiones

Para cada petición entrante, el trabajador:
1. Hace coincidir la clave `"METHOD /path"` con la tabla de rutas (sin distinción de mayúsculas/minúsculas).
2. Construye un objeto JSON `{ method, url, body, ip, headers }` como `Value::Json`.
3. Almacena la `tiny_http::Request` en `Arc<Mutex<Option<tiny_http::Request>>>` y la pasa a un `Executor` nuevo.
4. Ejecuta el fiber manejador coincidente de forma síncrona en el `Executor` de ese trabajador.
5. Cuando el manejador llama a `net.respond(...)`, la VM ejecuta `HttpRespond` que envía la respuesta.
6. Si el manejador sale sin llamar a `net.respond`, el trabajador envía una respuesta de fallback `500`.

### Apagado Controlado

`SHUTDOWN` es un `pub static AtomicBool` en `vm.rs`. Un manejador de Ctrl+C en `main.rs` lo establece en `true`. Los trabajadores lo comprueban cada ciclo `recv_timeout(100ms)`. El hilo principal se bloquea en un bucle `sleep(500ms)` también sondeando `SHUTDOWN`. Una vez establecido, todos los bucles salen y el proceso termina limpiamente.

---

## Gestión de Memoria

- **Sin recolector de basura**. Conteo de referencias mediante `Arc` (con `inc_ref`/`dec_ref` manuales para valores puntero con NaN-boxing).
- **Valores escalares** (Int, Float, Bool, Date, índice de función): almacenados completamente en el `u64` — cero asignaciones en heap.
- **Valores de colección**: `Arc<RwLock<T>>` proporciona propiedad compartida. Clonar un `Value` de colección solo incrementa el contador `Arc`.
- **Mutaciones**: `.insert()`, `.update()`, `.delete()` adquieren un bloqueo de escritura. Todos los handles a la misma colección ven el cambio.
- **Métodos de solo lectura**: `.size()`, `.get()`, `.contains()` adquieren un bloqueo de lectura — se permiten múltiples lectores concurrentes.

---

## Optimizaciones de Bucles

Estos opcodes son emitidos por el compilador para fusionar patrones comunes de contador de bucle en instrucciones únicas, reduciendo la sobrecarga de despacho y mejorando la trazabilidad del JIT:

| OpCode | Descripción |
|---|---|
| `IncLocal { reg }` | Incrementa el entero en el registro `reg` en 1 |
| `IncVar { idx }` | Incrementa el global en `idx` en 1 |
| `LoopNext { reg, limit_reg, target }` | Incrementa `reg`, salta a `target` si `reg <= limit_reg` |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target }` | Incrementa `inc_reg` (contador separado, p.ej. índice de array), incrementa `reg` (var de bucle), salto condicional |
| `IncVarLoopNext { g_idx, reg, limit_reg, target }` | Como `IncLocalLoopNext` pero `g_idx` es un contador global |
| `ArrayLoopNext { idx_reg, size_reg, target }` | Iteración de índice de array combinada |

### Transformación de Optimización

El compilador comprueba la última instrucción emitida antes del final de un paso de bucle. Si es `IncVar` o `IncLocal`, los reemplaza con `IncVarLoopNext` o `IncLocalLoopNext` respectivamente, fusionando el incremento con la prueba de bucle condicional:

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