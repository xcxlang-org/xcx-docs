> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/compiler/semantics.md).

# Análisis Semántico de XCX (Sema) — v3.1

La fase Sema valida el AST para corrección lógica y consistencia de tipos antes de la generación de bytecode. Consta de dos componentes: la **Tabla de Símbolos** y el **Verificador de Tipos**.

## Interner de Cadenas (`src/sema/interner.rs`)

El `Interner` mapea `&str → StringId(u32)`. Es la única fuente de verdad para todas las identidades de cadenas en el compilador. Se crea durante el lexing/parsing y se pasa por referencia al checker y al compilador.

```
Interner::intern("foo") → StringId(42)
Interner::lookup(StringId(42)) → "foo"
```

## Tabla de Símbolos (`src/sema/symbol_table.rs`)

La `SymbolTable` gestiona las ligaduras de variables en ámbitos anidados mediante una **cadena enlazada de punteros a padres**.

### Estructura

```rust
pub struct SymbolTable<'a> {
    parent: Option<&'a SymbolTable<'a>>,  // referencia al ámbito contenedor
    scopes: Vec<HashMap<String, Type>>,   // pila de marcos de ámbito (locales a esta tabla)
    consts: Vec<HashSet<String>>,         // qué nombres son const, por marco de ámbito
}
```

### Creación de un Ámbito Hijo

Al entrar en el cuerpo de una función o fiber, se crea una nueva `SymbolTable` con una referencia a la tabla contenedora en lugar de clonarla:

```rust
let mut func_symbols = SymbolTable::new_with_parent(symbols);
```

Esto es O(1) — solo se asigna un único `HashMap` vacío para el nuevo marco. La búsqueda asciende por la cadena de padres según sea necesario.

### Ciclo de Vida del Ámbito

- `enter_scope()` / `exit_scope()` insertan/eliminan marcos locales dentro de una tabla — se usan para cuerpos de `if`, `while`, `for`.
- `new_with_parent(parent)` crea una tabla hija enlazada al padre — se usa para cuerpos de función y fiber.
- `define(name, ty, is_const)` siempre escribe en el ámbito **más interno** (actual) de la tabla actual.
- `lookup(name)` recorre los ámbitos locales de más interno a más externo, luego sigue la cadena de punteros `parent`.
- `has_in_current_scope(name)` comprueba solo el marco más interno de la tabla actual — se usa para detectar redefiniciones dentro del mismo bloque.
- `is_const(name)` encuentra el ámbito propietario de `name`, luego comprueba `consts[scope_index]`.

### Importante: Sin Sombreado de Variables

XCX **no** soporta el sombreado de variables. Definir una variable que ya existe en el **ámbito actual** genera `RedefinedVariable`. Las variables en ámbitos padre son accesibles pero no pueden redeclararse en un ámbito hijo con el mismo nombre.

## Verificador de Tipos (`src/sema/checker.rs`)

La estructura `Checker` recorre el AST y acumula valores `TypeError`. El programa se compila únicamente si el `Vec<TypeError>` resultante está vacío.

### Estado del Checker

```rust
pub struct Checker<'a> {
    interner:       &'a Interner,
    loop_depth:     usize,
    functions:      HashMap<String, FunctionSignature>,
    fiber_context:  Option<Option<Type>>,
    is_table_lambda: bool,
    fiber_has_yield: bool,
}
```

### Pasada de Pre-escaneo (`pre_scan_stmts`)

Antes de verificar cualquier cuerpo de sentencia, el checker realiza un **escaneo de declaraciones anticipadas** de todos los nodos `FunctionDef` y `FiberDef` en la **lista de sentencias actual** (no recursivo). Esto permite que funciones y fibers sean llamadas antes de su definición en el archivo fuente (recursión mutua, llamada antes de declaración).

Para cada función/fiber encontrada, el checker registra:
- Un `FunctionSignature { params: Vec<Type>, return_type: Option<Type>, is_fiber: bool }` en `self.functions`
- Una entrada en la `SymbolTable` con `Type::Unknown` (para funciones) o `Type::Fiber(...)` (para fibers)

Las funciones de casting integradas `i`, `f`, `s`, `b` se pre-registran con parámetro `Type::Unknown` y tipos de retorno correspondientes.

### Banderas de Contexto

| Campo | Propósito |
|---|---|
| `loop_depth: usize` | Rastrea la profundidad de anidamiento de `while`/`for`. Cero → `break`/`continue` son errores. Se resetea a 0 al entrar en un cuerpo de fiber. |
| `fiber_context: Option<Option<Type>>` | `None` = no está en fiber; `Some(None)` = fiber void; `Some(Some(T))` = fiber tipado que produce `T`. |
| `fiber_has_yield: bool` | Se activa cuando se encuentra un `yield`. Se guarda/restaura en definiciones de fiber anidadas. |
| `is_table_lambda: bool` | Se activa dentro de predicados `.where()`; permite nombres de columna desnudos como identificadores buscándolos en `__row_tmp`. |

### Reglas de Inferencia de Tipos

- Los tipos de expresión se infieren de abajo hacia arriba a partir de literales y se propagan a través de operadores.
- `Type::Unknown` actúa como comodín — cualquier operación con `Unknown` pasa sin error.
- `Type::Json` es compatible con cualquier tipo en asignaciones y comparaciones.
- Promoción numérica: `Int op Float → Float`.
- Un literal de array vacío `[]` hereda su tipo del contexto de asignación si está disponible.
- `Type::Table([])` (lista de columnas vacía) es compatible con cualquier `Table(cols)` — la información de columnas se propaga de vuelta al tipo declarado de la variable tras la inferencia.

### `is_compatible(expected, actual) -> bool`

Reglas de compatibilidad clave:

| Regla | Descripción |
|---|---|
| Cualquiera es `Unknown` | Siempre compatible |
| Cualquiera es `Json` | Siempre compatible |
| `Int` ↔ `Float` | Mutuamente compatibles (promoción numérica) |
| `Int` ↔ `Date` | Compatible (los timestamps son enteros) |
| `Set(X)` ↔ `Array(inner)` | Compatible cuando el tipo de elemento interno coincide |
| `Set(N)` ↔ `Set(Z)` | Compatible (ambos conjuntos de tipo entero) |
| `Set(S)` ↔ `Set(C)` | Compatible (ambos conjuntos de tipo cadena) |
| `Table([])` ↔ `Table(cols)` | Compatible cuando cualquiera de las listas de columnas está vacía |
| `Table(a)` ↔ `Table(b)` | Compatible cuando las longitudes coinciden y todos los tipos de columna son compatibles por pares |
| `Map(k1,v1)` ↔ `Map(k2,v2)` | Comprobado recursivamente |
| `Fiber(None)` ↔ `Fiber(None)` | Ambos fibers void |
| `Fiber(Some(T))` ↔ `Fiber(Some(U))` | Compatible si y solo si T y U son compatibles |

### Verificación de Llamadas a Función

Para llamadas a función con nombre (`ExprKind::FunctionCall`):
1. Se busca primero en `self.functions` (para funciones/fibers declarados).
2. Si no se encuentra, se busca en la tabla de símbolos (para valores de función de primera clase, llamadas dinámicas).
3. Si la firma resuelta es un fiber, la llamada devuelve `Type::Fiber(Some(ret))` (instanciación de fiber, no invocación directa).
4. Los argumentos adicionales más allá del número de parámetros declarados se verifican pero no se rechazan (tolerancia variádica).
5. Las llamadas no resueltas insertan `UndefinedVariable`.

Para llamadas a función a nivel de sentencia (`StmtKind::FunctionCallStmt`):
- Discrepancia en el conteo de argumentos → `Other("Function expects N arguments, got M")`.
- Nombre desconocido no presente en la tabla de símbolos → `UndefinedVariable`.

### Verificación de Declaraciones de Fiber (`FiberDecl`)

1. Se busca `fiber_name` en `self.functions` (preferido) o en la tabla de símbolos.
2. Se verifica que la firma resuelta tenga `is_fiber: true`.
3. Se comprueban los tipos de argumentos contra los parámetros.
4. Se define la nueva variable con `Type::Fiber(inner_type)`.

### Verificación de `.where()` en Tablas

Cuando se verifica `table.where(predicate)`:
1. Se abre un ámbito temporal y se define `__row_tmp: Table(cols)`.
2. `is_table_lambda` se establece en `true`.
3. `collect_pred_idents()` recopila todos los nombres `Identifier` usados en el predicado.
4. Si algún identificador existe tanto en el ámbito externo **como** coincide con un nombre de columna → `S301 WherePredicateNameCollision`.
5. El predicado se verifica en cuanto a tipos; debe devolver `Bool`.
6. Se sale del ámbito y `is_table_lambda` se restaura.

### Verificación de `Table.join()`

El checker combina las definiciones de columnas: las columnas de la tabla izquierda más las columnas de la tabla derecha que no estén ya presentes (por igualdad de `StringId`). Si hay conflicto en el nombre de una columna, se conserva la de la tabla derecha. El tipo resultante es `Type::Table(combined_cols)`.

### Verificación de Bucles `for`

El campo `iter_type` de `ForIterType` se muta durante la verificación según el tipo inferido de `start`:
- `Type::Array(_)` → establece `ForIterType::Array`, la variable de bucle obtiene el tipo interno
- `Type::Set(st)` → establece `ForIterType::Set`, la variable de bucle obtiene el tipo de elemento del conjunto
- `Type::Table(cols)` → tratado como `ForIterType::Array`, la variable de bucle obtiene `Type::Table(cols)`
- `Type::Fiber(inner)` → establece `ForIterType::Fiber`, la variable de bucle obtiene el tipo de yield interno
- `Type::Int` (con palabra clave `to`) → permanece `ForIterType::Range`, la variable de bucle es `Int`

---

## Códigos de Error Validados

| Código | Condición |
|---|---|
| `UndefinedVariable(name)` | Nombre usado antes de su declaración |
| `RedefinedVariable(name)` | Nombre declarado dos veces en el mismo ámbito |
| `ConstReassignment(name)` | Asignación a una variable `const` |
| `TypeMismatch { expected, actual }` | El tipo de la expresión no coincide con el esperado |
| `InvalidBinaryOp { op, left, right }` | Operador usado con tipos incompatibles |
| `BreakOutsideLoop` | `break` fuera de `while`/`for` |
| `ContinueOutsideLoop` | `continue` fuera de `while`/`for` |
| `[S208] YieldOutsideFiber` | `yield` usado fuera de cualquier cuerpo de fiber |
| `[S209] FiberTypeMismatch` | `yield expr;` dentro de un fiber void (debería ser `yield;`) |
| `[S210] ReturnTypeMismatchInFiber` | `return;` desnudo en un fiber tipado (falta valor de retorno) |
| `[S301] WherePredicateNameCollision` | El nombre de variable local conflicta con un nombre de columna de tabla en `.where()` |
| `Other(msg)` | Errores contextuales varios (conteo de argumentos, método desconocido, iteración sobre fiber void, etc.) |

---

## Reporte de Errores

Los valores `TypeError` llevan un `Span { line, col, len }`. Tras la verificación, `main.rs` pasa cada error a `Reporter::error()`, que imprime:
1. `ERROR: <mensaje>`
2. La línea fuente relevante con número de línea
3. Un subrayado `~~~` comenzando en `col` con longitud `len`

La compilación se detiene inmediatamente tras el reporte de errores — no se genera bytecode si existe algún `TypeError`.