# Expansor de XCX — Documentación

> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/compiler/expander.md).

> **Archivo:** `src/parser/expander.rs`  
> Se ejecuta **después** del análisis, **antes** del análisis semántico.

---

## Tabla de Contenidos

1. [Descripción General](#descripción-general)
2. [Responsabilidades](#responsabilidades)
3. [Resolviendo include](#resolviendo-include)
4. [Prefijo de Alias](#prefijo-de-alias)
5. [Nombres Protegidos](#nombres-protegidos)
6. [Orden de Búsqueda de Rutas de Include](#orden-de-búsqueda-de-rutas-de-include)

---

## Descripción General

El Expansor es un paso de reescritura AST separado que se ejecuta después de la fase de análisis y antes del análisis semántico. Procesa directivas `include` e `include ... as alias`.

```rust
pub struct Expander<'a> {
    interner:       &'a mut Interner,
    visiting_files: HashSet<PathBuf>,   // para detección de dependencia circular
    included_files: HashSet<PathBuf>,   // deduplicación (cada archivo una sola vez)
    aliases:        HashMap<StringId, String>,
    include_paths:  Vec<PathBuf>,       // rutas de búsqueda adicionales
}
```

---

## Responsabilidades

### 1. Resolviendo include
`include "file.xcx";` se reemplaza por el AST insertado de ese archivo.

- Las dependencias circulares se detectan mediante `visiting_files: HashSet<PathBuf>`
- Los archivos se deduplican mediante `included_files: HashSet<PathBuf>` (cada archivo incluido una sola vez, a menos que tenga alias)

### 2. Prefijo de Alias
`include "math.xcx" as math;` hace que todos los nombres de nivel superior de ese archivo reciban el prefijo `math.name`. Los sitios de llamada (`math.sin(x)`) se reescriben de `MethodCall` a `FunctionCall { name: "math.sin" }` mediante `expand_expr_inplace`.

Las funciones `prefix_program()` / `prefix_stmt_impl()` / `prefix_expr_impl()` atraviesan todo el sub-AST, renombrando todas las referencias de símbolos de nivel superior.

### 3. Prefijo de Nombre de Fibra
Las referencias `FiberDecl::fiber_name` también tienen prefijo, para que las instanciaciones de fibras renombradas se resuelvan correctamente después del prefijo.

### 4. Prefijo de YieldFrom
Las expresiones `StmtKind::YieldFrom` se atraviesan de modo que las llamadas del constructor de fibras dentro de `yield from` también se renombren.

---

## Resolviendo include

```xcx
include "utils.xcx";           --- Include simple (deduplicado)
include "math.xcx" as math;    --- Include con alias
```

Después de un include con alias:
- Todos los símbolos de nivel superior de `math.xcx` obtienen el prefijo `math.`
- Las llamadas a `math.sin(x)` se reescriben de `MethodCall` a `FunctionCall { name: "math.sin" }`

Ejemplo:

```xcx
--- math.xcx define:
func sin(f: x -> f) { ... }
func cos(f: x -> f) { ... }

--- Después de include "math.xcx" as math:
--- sin → math.sin
--- cos → math.cos
--- Llamada math.sin(3.14) → FunctionCall("math.sin", [3.14])
```

---

## Prefijo de Alias

El algoritmo `prefix_stmt_impl` / `prefix_expr_impl`:

1. Recopila todos los nombres de nivel superior del programa (`top_level_names: HashSet<StringId>`)
2. Para cada declaración/expresión: si el identificador pertenece a `top_level_names`, se reemplaza con `prefix.name`
3. Maneja específicamente:
   - `FiberDecl::fiber_name` — para que las instanciaciones funcionen después del renombrado
   - `StmtKind::YieldFrom` — expresiones de fibra en `yield from`
   - `MethodCall` en un objeto con alias → reescrito a `FunctionCall`

---

## Nombres Protegidos

Los siguientes nombres **nunca** tienen prefijo (built-ins protegidos):

```
json    date    store   halt    terminal
net     env     crypto  EMPTY   math
random  i       f       s       b
from    main
```

---

## Orden de Búsqueda de Rutas de Include

1. Relativo al directorio del archivo actual
2. En el directorio `lib/` (relativo al CWD, luego caminando hacia arriba desde la ruta del ejecutable)

Se pueden agregar rutas adicionales mediante:

```rust
expander.add_include_path(path);
```

En `main.rs`, el directorio `lib/` relativo al CWD se agrega automáticamente:

```rust
if let Ok(cwd) = std::env::current_dir() {
    let lib_path = cwd.join("lib");
    if lib_path.exists() {
        expander.add_include_path(lib_path);
    }
}
```

---

## Detección de Errores

| Error | Descripción |
|---|---|
| `Circular dependency` | Bucle de include detectado (`visiting_files`) |
| `File not found` | El archivo `include` no existe en ninguna de las rutas de búsqueda |
| `Could not read file` | Error de E/S al leer el archivo |
