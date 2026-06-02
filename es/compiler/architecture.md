> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/compiler/architecture.md).

# Arquitectura del Compilador XCX — v3.1

El compilador XCX está implementado en Rust y sigue una arquitectura de canalización multi-etapa.

## Canalización de Compilación

```
Código Fuente
    │
    ▼
1. Escáner Léxico (Scanner)   — src/lexer/scanner.rs
    │  Produce: Flujo de tokens
    ▼
2. Analizador (Pratt)          — src/parser/pratt.rs
    │  Produce: AST bruto (Program)
    ▼
3. Expandidor                  — src/parser/expander.rs
    │  Produce: AST expandido (directivas include resueltas, aliases prefijados)
    ▼
4. Verificador de Tipos (Sema) — src/sema/checker.rs
    │  Produce: AST validado, anotado
    ▼
5. Compilador (Backend)        — src/backend/mod.rs
    │  Produce: FunctionChunk (main) + Arc<Vec<Value>> constantes + Arc<Vec<FunctionChunk>> funciones
    ▼
6. VM                          — src/backend/vm.rs
    │  Ejecuta bytecode basado en registros
    │  Bucles calientes detectados → Grabación de rastreo comienza
    ▼
7. JIT (Cranelift)             — src/backend/jit.rs
       Compila rastreos grabados en código máquina nativo
```

> **Nota**: El Expandidor es parte del módulo `src/parser/` pero se ejecuta como una fase de post-análisis distinta, antes del análisis semántico. El JIT es una capa de aceleración opcional que se activa automáticamente para bucles calientes — la ejecución de bytecode continúa sin interrupciones si un rastreo no está compilado aún.

## Estructura del Proyecto

```
src/
├── lexer/
│   ├── scanner.rs      # Escáner a nivel de byte (&[u8])
│   └── token.rs        # Definiciones TokenKind y Span
├── parser/
│   ├── pratt.rs        # Analizador Pratt (flujo de tokens → AST)
│   ├── expander.rs     # Resolución de include y prefijación de alias
│   └── ast.rs          # Definiciones de nodos AST (Expr, Stmt, Type, Program)
├── sema/
│   ├── checker.rs      # Verificador de tipos y resolutor de variables
│   ├── symbol_table.rs # Tabla de símbolos de ámbito jerárquico con cadena de punteros padre
│   └── interner.rs     # Internador de cadenas (str → StringId)
├── backend/
│   ├── mod.rs          # Compilador de bytecode (AST → OpCode) con asignador de registros
│   ├── vm.rs           # VM basada en registros con Values en caja NaN + ganchos JIT de rastreo
│   ├── jit.rs          # Compilador de rastreo basado en Cranelift
│   └── repl.rs         # REPL interactivo
└── diagnostic/
    └── report.rs       # Reportero de errores con resaltado de fuente
```

## Sistema de Diagnóstico

El compilador utiliza una estructura `Reporter` para producir mensajes de error contextuales. Cada error incluye:
- **Nivel**: Variante ERROR o HALT
- **Ubicación**: Número de línea y columna
- **Resaltado visual**: La línea de fuente relevante con un subrayado `~~~`

Los errores semánticos (`TypeError`) se recopilan en un `Vec` durante la fase de verificación y se informan todos a la vez antes de que comience la generación de bytecode. Si existen errores, la compilación se detiene en ese punto.

Los errores de tiempo de ejecución producidos por la VM incluyen información de ubicación de fuente (línea y columna) derivada de la tabla `spans` almacenada junto con el bytecode de cada `FunctionChunk`.

## Decisiones de Diseño Clave

### Valores en Caja NaN
Todos los valores se representan como un único `u64` usando NaN-boxing. Los bits altos de un NaN tranquilo IEEE 754 se utilizan como etiquetas de tipo, y los 48 bits bajos llevan la carga útil (entero, booleano o puntero). Los flotantes se almacenan tal cual. Esto significa que cada `Value` tiene exactamente 8 bytes — sin asignación de montón para escalares, sin overhead de etiqueta en enum, sin indirección de puntero para primitivos. Consulte `src/backend/vm.rs` para el diseño completo de etiquetas.

### VM Basada en Registros
La VM es **basada en registros**, no basada en pila. Cada marco de función posee un `Vec<Value>` plano indexado por número de ranura. Los opcodes hacen referencia a registros de origen y destino directamente (campos `dst`, `src1`, `src2`). El `FunctionCompiler` del compilador mantiene un simple asignador bump (`push_reg()` / `pop_reg()`) para registros temporales. Los locales y temporales viven en el mismo array plano — no hay pila de operandos separada.

### JIT de Rastreo (Cranelift)
Los saltos hacia atrás calientes se detectan mediante un contador `hot_counts: Vec<usize>` por marco. Cuando un borde de retorno de bucle alcanza el umbral (50 iteraciones), comienza una fase de grabación de rastreo. El `Executor` registra variantes `TraceOp` — versiones tipadas y especializadas de ops del intérprete — hasta que el rastreo se cierra (bucle retorna al IP inicial). El `Trace` completado se compila en código nativo via Cranelift y se cachea en `trace_cache`. Las iteraciones posteriores de ese bucle omiten el intérprete completamente. Los guardias (`GuardInt`, `GuardFloat`, `GuardTrue`, `GuardFalse`) en el rastreo manejan especialización de tipo; una falla del guardia causa una salida lateral de vuelta al intérprete en el IP correcto.

### Internación de Cadenas
Todos los identificadores y literales de cadena se internan via `Interner` en `StringId` (u32). Esto evita asignaciones de cadena repetidas y comparaciones de montón en toda la canalización.

### Deduplicación de Constantes
El compilador mantiene un `string_constants: HashMap<String, usize>` que asegura que cada valor de cadena único se almacene solo una vez en la tabla de constantes. Esto es especialmente efectivo para nombres de métodos integrados emitidos frecuentemente durante la compilación.

### Despacho de Método via Enum
Las llamadas de método integrado se compilan en `OpCode::MethodCall { kind: MethodKind, base, arg_count }` donde `MethodKind` es una enumeración `Copy` que cubre ~50 métodos integrados. Los nombres de método desconocidos o dinámicos (por ejemplo, acceso de campo JSON) utilizan la ruta separada `OpCode::MethodCallCustom { method_name_idx, base, arg_count }`. Esto elimina comparaciones de cadena en el bucle de despacho VM.

### Compilación de Dos Pasos
El backend realiza un primer paso (`register_globals_recursive`) para asignar índices a todos los globales, funciones y fibras antes de emitir cualquier bytecode.

### Bytecode Anotado con Span
Cada opcode emitido se empareja con un `Span` del AST de fuente. `FunctionChunk` almacena `spans: Arc<Vec<Span>>` junto a `bytecode: Arc<Vec<OpCode>>`. La VM utiliza esto para producir mensajes de error de tiempo de ejecución precisos por línea.

### Fibra como Corutina
Las fibras no son hilos del SO. Cada `FiberState` almacena su propio `ip` y `locals`, reanudados cooperativamente por la VM. En yield, los locals se mueven de vuelta a `FiberState`; en reanudación, se mueven de nuevo. Esto es libre de asignación más allá de la creación inicial del `Vec`.

### Representación de Valor Thread-Safe
Todas las colecciones mutables compartidas utilizan `Arc<RwLock<T>>` (via `parking_lot`). El bytecode y spans de `FunctionChunk` se envuelven en `Arc<Vec<...>>` para que puedan compartirse entre hilos de trabajo HTTP sin copiar. `Value` es `Copy` + `Send` + `Sync`.

### Apagado Graceful de HTTP
Un `AtomicBool` global (`SHUTDOWN`) se establece mediante un controlador de señal Ctrl+C (via la crate `ctrlc`). Todos los hilos de trabajo HTTP y el bucle de despacho principal colean esta bandera y salen limpiamente antes de que el proceso termine.

### Tabla de Símbolos de Cadena de Ámbito
El `SymbolTable` utiliza una cadena de punteros padre en lugar de clonar profundamente. Entrar en un ámbito de función crea un nuevo `SymbolTable` con una referencia al padre — la búsqueda camina la cadena hacia arriba. La entrada de ámbito de función es O(1) en lugar de O(n).

### Escáner a Nivel de Byte
El escáner léxico funciona en `&[u8]` (una referencia a los bytes de fuente originales) sin asignar un `Vec<char>`. El manejo Unicode se realiza solo donde es necesario. La detección de comentarios usa `slice.starts_with(b"---")`.
