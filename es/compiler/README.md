> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/compiler/README.md).

# Compilador XCX — Documentación v3.1

> Compilador del lenguaje XCX implementado en Rust. Canalización multi-etapa: lexer → parser → análisis semántico → compilador de bytecode → máquina virtual → JIT (Cranelift).

---

## Índice de Contenidos

1. [Descripción General de Arquitectura](#descripción-general-de-arquitectura)
2. [Inicio Rápido](#inicio-rápido)
3. [Estructura del Proyecto](#estructura-del-proyecto)
4. [Canalización de Compilación](#canalización-de-compilación)
5. [Módulos](#módulos)
6. [Decisiones de Diseño Clave](#decisiones-de-diseño-clave)
7. [Sistema de Diagnóstico](#sistema-de-diagnóstico)
8. [Seguridad](#seguridad)

---

## Descripción General de Arquitectura

```
Código Fuente (.xcx)
        │
        ▼
  1. Lexer         src/lexer/scanner.rs       → flujo de tokens
        │
        ▼
  2. Parser        src/parser/pratt.rs        → AST bruto (Program)
        │
        ▼
  3. Expander      src/parser/expander.rs     → AST expandido (include, aliases)
        │
        ▼
  4. Sema          src/sema/checker.rs        → AST validado, anotado
        │
        ▼
  5. Compilador    src/backend/mod.rs         → FunctionChunk + constantes + funciones
        │
        ▼
  6. VM            src/backend/vm.rs          → ejecución de bytecode (basada en registros)
        │  bucles calientes → grabación de traza
        ▼
  7. JIT           src/backend/jit.rs         → código máquina nativo (Cranelift)
```

---

## Inicio Rápido

```bash
# Iniciar REPL
xcx

# Ejecutar archivo
xcx programa.xcx

# Versión
xcx --version

# Ayuda
xcx --help
```

**Dentro del REPL:**

```
xcx> !help     # mostrar ayuda
xcx> !clear    # limpiar pantalla
xcx> !exit     # salir
```

---

## Estructura del Proyecto

```
src/
├── lexer/
│   ├── scanner.rs        # Escáner de bytes (&[u8])
│   └── token.rs          # TokenKind y Span
├── parser/
│   ├── pratt.rs          # Parser Pratt (tokens → AST)
│   ├── expander.rs       # Resolución de include y prefijación de aliases
│   └── ast.rs            # Definiciones de nodos AST (Expr, Stmt, Type, Program)
├── sema/
│   ├── checker.rs        # Verificación de tipos y resolución de variables
│   ├── symbol_table.rs   # Tabla de símbolos jerárquica
│   └── interner.rs       # Internador de cadenas (str → StringId)
├── backend/
│   ├── mod.rs            # Compilador de bytecode (AST → OpCode)
│   ├── vm.rs             # VM basada en registros con NaN-boxing + hooks JIT
│   ├── jit.rs            # Compilador de traza Cranelift
│   └── repl.rs           # REPL interactivo
└── diagnostic/
    └── report.rs         # Reportador de errores con resaltado de código
```

---

## Canalización de Compilación

### 1. Lexer
Convierte bytes fuente en tokens. Opera sobre `&[u8]` — sin asignación de `Vec<char>`.

### 2. Parser
El Parser Pratt (Precedencia de Operadores de Arriba hacia Abajo) construye el AST. Anticipación de un token. Recuperación de errores mediante `synchronize()`.

### 3. Expander
Se ejecuta **después** del análisis, **antes** del análisis semántico. Resuelve directivas `include` y prefija nombres de alias.

### 4. Análisis Semántico
Comprueba tipos, detecta variables no definidas, valida contexto de fiber/bucle. Recopila todos los errores antes de generar bytecode.

### 5. Compilador
Dos pasadas. Pasada 1: registra globales/funciones. Pasada 2: emite bytecode con anotaciones de Span.

### 6. VM
Máquina virtual basada en registros. Cada frame tiene un `Vec<Value>` plano indexado por números de slot `u8`. Valores de 8 bytes (NaN-boxing).

### 7. JIT
Compilación automática de bucles calientes a código nativo mediante Cranelift. Transparente para el desarrollador.

---

## Módulos

Descripciones detalladas en archivos separados:

- [`docs/lexer.md`](docs/lexer.md) — Lexer / Scanner
- [`docs/parser.md`](docs/parser.md) — Parser Pratt y AST
- [`docs/expander.md`](docs/expander.md) — Expander
- [`docs/sema.md`](docs/sema.md) — Análisis Semántico
- [`docs/backend.md`](docs/backend.md) — Compilador y VM
- [`docs/jit.md`](docs/jit.md) — JIT Cranelift
- [`docs/language.md`](docs/language.md) — Referencia del Lenguaje XCX

---

## Decisiones de Diseño Clave

### NaN-Boxing de valores
Cada valor es un único `u64` (8 bytes). Los bits altos del NaN silencioso de IEEE 754 sirven como etiquetas de tipo. Los 48 bits inferiores llevan el payload. Sin asignación en heap para escalares, sin sobrecarga de etiqueta en enums, sin indirección de puntero para primitivos.

### VM basada en registros
En lugar de una pila de operandos — un array plano `Vec<Value>` por frame. Los opcodes referencian directamente los registros fuente/destino. Asignación de registros simple con puntero de incremento en el compilador.

### JIT de rastreo (Cranelift)
Los saltos hacia atrás (arcos de bucle) se cuentan por IP. Tras alcanzar un umbral (50 iteraciones), comienza el registro de trazas. La traza terminada se compila a código nativo por Cranelift. Las guardas de tipo (`GuardInt`, `GuardFloat`) gestionan la especialización de tipos; una guarda fallida activa el retroceso al intérprete.

### Internado de cadenas
Todos los identificadores y literales de cadena son internados por el `Interner` a `StringId (u32)`. Elimina asignaciones repetitivas de cadenas y comparaciones en heap a lo largo de todo el pipeline.

### Compilación en dos pasadas
La pasada 1 (`register_globals_recursive`) asigna índices a todos los globales y funciones antes de emitir cualquier bytecode — habilitando la recursión mutua y las llamadas antes de la declaración.

---

## Sistema de Diagnóstico

El `Reporter` en `src/diagnostic/report.rs` produce mensajes de error contextuales:

```
ERROR: [S101] Undefined variable: foo
   12 | i: x = foo + 1;
              ~~~
```

Cada error contiene:
- **Nivel**: ERROR o HALT
- **Ubicación**: número de línea y columna
- **Resaltado visual**: línea fuente relevante con subrayado `~~~`

Los errores semánticos (`TypeError`) se recopilan en un `Vec` durante la fase de verificación y se reportan todos a la vez antes de la generación de bytecode.

---

## Seguridad

### Protección SSRF en Red
Objetivos bloqueados:
- URLs `file://`
- `169.254.x.x` (link-local / endpoint de metadatos AWS)
- Rangos privados: `10.x`, `192.168.x`, `172.16–31.x` (excluyendo localhost)

Se aplica tanto a `HttpCall` como a `HttpRequest`.

### Límite de Tamaño de Contenido HTTP
En `HttpServe`, el contenido de la respuesta se comprueba tras `into_string()`. Si supera **10 MB**, se devuelve un error JSON 413 en lugar del contenido real.

### Cabeceras CORS
Todas las respuestas de `HttpServe` incluyen automáticamente:
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

Las peticiones de preflight `OPTIONS` reciben una respuesta `204` sin llamar al manejador.

### Rutas de Archivo Seguras
Las operaciones `store.*` validan las rutas — bloqueando `..`, rutas absolutas y letras de unidad de Windows.