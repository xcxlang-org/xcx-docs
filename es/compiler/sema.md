> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/compiler/sema.md).

# Análisis Semántico de XCX (Sema) — Documentación

> **Archivos:** `src/sema/checker.rs`, `src/sema/symbol_table.rs`, `src/sema/interner.rs`

---

## Tabla de Contenidos

1. [Descripción General](#descripción-general)
2. [Interner de Cadenas](#interner-de-cadenas)
3. [Tabla de Símbolos](#tabla-de-símbolos)
4. [Verificador de Tipos](#verificador-de-tipos)
5. [Reglas de Compatibilidad de Tipos](#reglas-de-compatibilidad-de-tipos)
6. [Códigos de Error](#códigos-de-error)
7. [Reporte de Errores](#reporte-de-errores)

---

## Descripción General

La fase Sema valida el AST para corrección lógica y consistencia de tipos antes de la generación de bytecode. Consta de tres componentes:

```
Interner → StringId (u32)
     ↓
SymbolTable → ámbitos de tipos jerárquicos
     ↓
Checker → colección de TypeErrors
```

El programa se compila únicamente si el `Vec<TypeError>` resultante está vacío.

---

## Interner de Cadenas

**Archivo:** `src/sema/interner.rs`

El `Interner` mapea `&str → StringId(u32)`. Es la única fuente de verdad para todas las identidades de cadenas en el compilador. Se crea durante el lexing/parsing y se pasa por referencia al checker y al compilador.

```rust
Interner::intern("foo") → StringId(42)
Interner::lookup(StringId(42)) → "foo"
```

**Implementación:**

```rust
pub struct Interner {
    map:     HashMap<String, StringId>,
    strings: Vec<String>,
}
```

Cada cadena única se almacena una sola vez en `strings: Vec<String>`. El resto del pipeline trabaja con IDs numéricos, eliminando comparaciones en el heap durante la verificación de tipos y la compilación.

---

## Tabla de Símbolos

**Archivo:** `src/sema/symbol_table.rs`

La `SymbolTable` gestiona las ligaduras de variables en ámbitos anidados mediante una **cadena de punteros a padres**.

### Estructura

```rust
pub struct SymbolTable<'a> {
    parent: Option<&'a SymbolTable<'a>>,  // ref al ámbito circundante
    scopes: Vec<HashMap<String, Type>>,   // pila de marcos de ámbito (locales)
    consts: Vec<HashSet<String>>,         // qué nombres son const, por marco
}
```

### Creación de un Ámbito Hijo

Al entrar en el cuerpo de una función o fiber, se crea una nueva `SymbolTable` con una referencia a la tabla circundante en lugar de clonarla:

```rust
let mut func_symbols = SymbolTable::new_with_parent(symbols);
```

Esto es O(1) — solo se asigna un `HashMap` vacío para el nuevo marco. Las búsquedas recorren la cadena de padres según sea necesario.

### Ciclo de Vida del Ámbito

| Método | Descripción |
|---|---|
| `enter_scope()` | Inserta un nuevo marco — para cuerpos de `if`, `while`, `for` |
| `exit_scope()` | Sale del ámbito actual |
| `new_with_parent(parent)` | Crea una tabla hija — para cuerpos de función/fiber |
| `define(name, ty, is_const)` | Siempre escribe en el ámbito **más interno** |
| `lookup(name)` | Recorre de adentro hacia afuera, luego al padre |
| `has_in_current_scope(name)` | Comprueba solo el marco más interno — para detección de redeclaración |
| `is_const(name)` | Comprueba si el ámbito propietario marcó el nombre como const |

### Sin Sombreado de Variables

XCX **no** soporta el sombreado de variables. Definir una variable que ya existe en el **ámbito actual** devuelve `RedefinedVariable`. Las variables en ámbitos padre son accesibles pero no pueden redeclararse en un ámbito hijo con el mismo nombre.

---

## Verificador de Tipos

**Archivo:** `src/sema/checker.rs`

La estructura `Checker` recorre el AST y acumula valores `TypeError`.

### Estado del Checker

```rust
pub struct Checker<'a> {
    interner:          &'a Interner,
    loop_depth:        usize,
    functions:         HashMap<String, FunctionSignature>,
    fiber_context:     Option<Option<Type>>,  // None=fuera de fiber, Some(None)=void, Some(Some(T))=tipado
    is_table_lambda:   bool,
    fiber_has_yield:   bool,
    in_yield_expr:     bool,
    last_expr_was_db_io: bool,
}
```

### Banderas de Contexto

| Campo | Propósito |
|---|---|
| `loop_depth: usize` | Rastrea la profundidad de anidamiento de `while`/`for`. Cero → `break`/`continue` son errores. Se resetea a 0 al entrar en un cuerpo de fiber. |
| `fiber_context: Option<Option<Type>>` | `None` = no está en fiber; `Some(None)` = fiber void; `Some(Some(T))` = fiber tipado que produce `T` |
| `fiber_has_yield: bool` | Se activa cuando se encuentra `yield`. Se guarda/restaura en definiciones de fiber anidadas. |
| `is_table_lambda: bool` | Se activa dentro de predicados `.where()`; permite nombres de columna desnudos como identificadores mediante `__row_tmp` |
| `in_yield_expr: bool` | Rastrea si estamos dentro de una expresión yield |
| `last_expr_was_db_io: bool` | Bandera para operaciones de E/S de base de datos |

### Pasada de Pre-escaneo

Antes de verificar cualquier cuerpo de sentencia, el checker realiza un **escaneo de declaraciones anticipadas** de todos los nodos `FunctionDef` y `FiberDef` en la **lista de sentencias actual** (no recursivamente). Esto permite que funciones y fibers sean llamadas antes de su definición en el archivo fuente (recursión mutua, llamada antes de declaración).

Para cada función/fiber encontrada, el checker registra:
- `FunctionSignature { params: Vec<Type>, return_type: Option<Type>, is_fiber: bool }` en `self.functions`
- Una entrada en la `SymbolTable` con `Type::Unknown` (para funciones) o `Type::Fiber(...)` (para fibers)

Las funciones de casting integradas `i`, `f`, `s`, `b` se pre-registran.

### Reglas de Inferencia de Tipos

- Los tipos de expresión se infieren de abajo hacia arriba a partir de literales y se propagan a través de operadores.
- `Type::Unknown` actúa como comodín — cualquier operación con `Unknown` pasa sin error.
- `Type::Json` es compatible con cualquier tipo en asignaciones y comparaciones.
- Promoción numérica: `Int op Float → Float`.
- Un literal de array vacío `[]` hereda su tipo del contexto de asignación si está disponible.
- `Type::Table([])` (lista de columnas vacía) es compatible con cualquier `Table(cols)`.

---

## Reglas de Compatibilidad de Tipos

La función `is_compatible(expected, actual) -> bool`:

| Regla | Descripción |
|---|---|
| Cualquiera es `Unknown` | Siempre compatible |
| Cualquiera es `Json` | Siempre compatible |
| `Int` ↔ `Float` | Mutuamente compatibles (promoción numérica) |
| `Int` ↔ `Date` | Compatible (los timestamps son enteros) |
| `Set(X)` ↔ `Array(inner)` | Compatible cuando el tipo de elemento coincide |
| `Set(N)` ↔ `Set(Z)` | Compatible (ambos conjuntos de tipo entero) |
| `Set(S)` ↔ `Set(C)` | Compatible (ambos conjuntos de tipo cadena) |
| `Table([])` ↔ `Table(cols)` | Compatible cuando cualquiera de las listas de columnas está vacía |
| `Table(a)` ↔ `Table(b)` | Compatible cuando las longitudes coinciden y los tipos de columna coinciden por pares |
| `Map(k1,v1)` ↔ `Map(k2,v2)` | Comprobado recursivamente |
| `Fiber(None)` ↔ `Fiber(None)` | Ambos fibers void |
| `Fiber(Some(T))` ↔ `Fiber(Some(U))` | Compatible si T y U son compatibles |

---

## Códigos de Error

| Código | Condición |
|---|---|
| `[S101] UndefinedVariable(name)` | Nombre usado antes de su declaración |
| `[S102] RedefinedVariable(name)` | Nombre declarado dos veces en el mismo ámbito |
| `[S103] TypeMismatch { expected, actual }` | El tipo de la expresión no coincide con el esperado |
| `[S104] InvalidBinaryOp { op, left, right }` | Operador usado con tipos incompatibles |
| `[S105] ConstReassignment(name)` | Asignación a una variable `const` |
| `[S106] BreakOutsideLoop` | `break` fuera de `while`/`for` |
| `[S107] ContinueOutsideLoop` | `continue` fuera de `while`/`for` |
| `[S108] IndexAccessNotSupported(type)` | Indexación de un tipo no soportado |
| `[S109] PropertyNotFound` | La propiedad no existe en el tipo |
| `[S110] MethodNotFound` | El método no existe en el tipo |
| `[S111] InvalidArgumentCount` | Número incorrecto de argumentos |
| `[S208] YieldOutsideFiber` | `yield` usado fuera del cuerpo de un fiber |
| `[S209] FiberTypeMismatch` | `yield expr;` dentro de un fiber void (debería ser `yield;`) |
| `[S210] ReturnTypeMismatchInFiber` | `return;` desnudo en un fiber tipado (falta valor de retorno) |
| `[S211] CannotIterateOverVoidFiber` | Iteración sobre un fiber void |
| `[S212] CannotRunTypedFiber` | Llamada a `.run()` en un fiber tipado |
| `[S301] WherePredicateNameCollision` | El nombre de variable local conflicta con el nombre de columna en `.where()` |
| `[S302] TableRowCountMismatch` | La fila de tabla tiene un número de columnas diferente al esquema |
| `[D401] Rule violation` | `remove()` sin `.where()` |
| `Other(msg)` | Errores contextuales varios (conteo de argumentos, método desconocido, etc.) |

---

## Reporte de Errores

Los valores `TypeError` llevan un `Span { line, col, len }`. Tras la verificación, `main.rs` pasa cada error a `Reporter::error()`, que imprime:

```
ERROR: [S101] Undefined variable: foo
   12 | i: x = foo + 1;
              ~~~
```

La compilación se detiene inmediatamente tras reportar errores — el bytecode no se genera si existe algún `TypeError`.

---

## Verificación de Llamadas a Función

Para llamadas a función con nombre (`ExprKind::FunctionCall`):
1. Se busca primero en `self.functions` (para funciones/fibers declarados)
2. Si no se encuentra, se busca en la tabla de símbolos (para valores de función de primera clase, llamadas dinámicas)
3. Si la firma resuelta es un fiber, la llamada devuelve `Type::Fiber(Some(ret))` (instanciación de fiber, no llamada directa)
4. Los argumentos adicionales más allá del número de parámetros declarados se verifican pero no se rechazan (tolerancia variádica)
5. Las llamadas no resueltas añaden un error `UndefinedVariable`

---

## Verificación de `.where()` en Tablas

Cuando se verifica `table.where(predicate)`:
1. Se abre un ámbito temporal y se define `__row_tmp: Table(cols)`
2. `is_table_lambda` se establece en `true`
3. `collect_pred_idents()` recopila todos los nombres `Identifier` usados en el predicado
4. Si algún identificador existe tanto en el ámbito externo **como** coincide con un nombre de columna → `S301 WherePredicateNameCollision`
5. El predicado se verifica; debe devolver `Bool`
6. El ámbito se cierra y `is_table_lambda` se restaura