# Analizador de XCX — v3.1

> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/compiler/parser.md).

El Analizador de XCX transforma el flujo de tokens en un Árbol de Sintaxis Abstracta (AST) de alto nivel.

## Arquitectura: Análisis Pratt

XCX utiliza un **Analizador Pratt** (Precedencia de Operadores de Arriba hacia Abajo).

- **Archivo**: `src/parser/pratt.rs`
- **Anticipación**: Un token (`current` + `peek`), avanzado manualmente con `advance()`.
- **Recuperación de Errores**: En error de sintaxis, `synchronize()` omite tokens hasta el siguiente punto y coma o una palabra clave conocida de inicio de declaración (`func`, `fiber`, `if`, `for`, `const`, `return`, `>!`, etc.).

La estructura `Parser` toma prestado la cadena de origen para la vida útil `'a`, y `Scanner<'a>` es parametrizado por la misma vida útil, reflejando el análisis basado en segmento de bytes.

### Niveles de Precedencia (menor → mayor)

| Nivel | Operadores |
|---|---|
| `Lowest` | — |
| `Lambda` | `->` |
| `Assignment` | `=` |
| `LogicalOr` | `OR`, `\|\|` |
| `LogicalAnd` | `AND`, `&&` |
| `Equals` | `==`, `!=` |
| `LessGreater` | `>`, `<`, `>=`, `<=`, `HAS` |
| `Sum` | `+`, `-`, `++` |
| `SetOp` | `UNION`, `INTERSECTION`, `DIFFERENCE`, `SYMMETRIC_DIFFERENCE` |
| `Product` | `*`, `/`, `%` |
| `Power` | `^` |
| `Prefix` | `-x` |
| `Call` | `.`, `[` |

## Despacho de Sentencias

`parse_statement_internal()` despacha según el token actual:

- **Palabras clave de tipo** (`i`, `f`, `s`, `b`, `array`, `set`, `map`, `date`, `table`, `json`) → `parse_var_decl()`, o `parse_assignment()` si va seguido de `=`
- **`const`** → `parse_var_decl()` con `is_const = true`
- **`var`** (identificador) → declaración de variable con tipo inferido
- **`>!`** → `parse_print_stmt()`
- **`>?`** → `parse_input_stmt()`
- **`halt`** → `parse_halt_stmt()`
- **`if`** → `parse_if_statement()`
- **`while`** → `parse_while_statement()`
- **`for`** → `parse_for_statement()`
- **`break`** / **`continue`** → `parse_break_statement()` / `parse_continue_statement()`
- **`func`** → `parse_func_def()`
- **`fiber`** → `parse_fiber_statement()` (despacha a def o decl según el peek)
- **`return`** → `parse_return_stmt()`
- **`yield`** → `parse_yield_stmt()` (gestiona `yield expr`, `yield from expr` y `yield;`)
- **`@wait`** → `parse_wait_stmt()`
- **`serve`** → `parse_serve_stmt()`
- **`net`** → `parse_net_stmt()`
- **`include`** → `parse_include_stmt()`
- **Identificador + `=`** → `parse_assignment()`
- **Identificador + `(`** → `parse_func_call_stmt()`
- **Cualquier otra cosa** → `parse_expr_stmt()`

## Estilos de Definición de Función

XCX soporta dos estilos sintácticamente distintos para definir funciones:

**Estilo de llaves** (similar a C):
```xcx
func name(i: x, s: y -> i) {
    return x + 1;
}
```

**Estilo XCX** (bloque con palabras clave):
```xcx
func:i: name(i: x, s: y) do;
    return x + 1;
end;
```

Ambos producen nodos AST `StmtKind::FunctionDef` idénticos. El tipo de retorno en el estilo de llaves se declara con `-> type` dentro de la lista de parámetros o después de `)`.

## Sentencias de Fiber

`parse_fiber_statement()` examina `peek` para decidir:
- `peek == Colon` → `parse_fiber_decl()` (instanciación: `fiber:T: varname = fiberDef(args);`)
- en caso contrario → `parse_fiber_def()` (definición: `fiber name(params) { body }`)

`parse_fiber_decl()` también gestiona el caso en que, tras analizar el tipo y el nombre, el token actual es `(` — en ese caso pivota a `finish_fiber_def()` (una definición con anotación de tipo `fiber:` al inicio).

## Construcciones Clave Analizadas

- **Declaraciones de variables**: `i: name = expr;`, `const s: NAME = expr;`, `var name = expr;`
- **Flujo de control**: `if (cond) then; ... elseif (cond) then; ... else; ... end;`
- **Bucle while**: `while (cond) do; ... end;`
- **Bucle for**: `for x in expr do; ... end;` y `for x in start to end @step n do; ... end;`
- **Funciones**: `func` (dos estilos, ver arriba)
- **Fibers**: `fiber name(params) { body }` y `fiber:T: varname = fiberName(args);`
- **Yield**: `yield expr;`, `yield from expr;`, `yield;`
- **HTTP**: `serve: name { port=..., routes=... };`, `net.get(url)`, `net.request { ... } as resp;`, `net.respond(status, body);`
- **Colecciones**: Array `[a, b, c]`, Set `set:N { 1,,10 }`, Map `[k :: v, ...]`, Table `table { columns=[...] rows=[...] }`
- **Bloques raw**: `<<<...>>>` para JSON/cadenas en línea
- **Include**: `include "path";` o `include "path" as alias;`
- **E/S**: `>! expr;` (imprimir), `>? varname;` (entrada)
- **Halt**: `halt.alert >! msg;`, `halt.error >! msg;`, `halt.fatal >! msg;`
- **Wait**: `@wait(ms);` o `@wait ms;`
- **Literales de fecha**: `date("2024-01-01")` o `date("01/01/2024", "DD/MM/YYYY")`

## Análisis de Expresiones

`parse_expression(precedence)` llama a `parse_prefix()` para el lado izquierdo, luego itera llamando a `parse_infix(left)` mientras la precedencia del token peek supere el mínimo actual.

Analizadores de prefijo clave:
- **Identificadores**: Si va seguido de `(`, se analiza como `FunctionCall`; de lo contrario como `Identifier`.
- **Literales**: `IntLiteral`, `FloatLiteral`, `StringLiteral`, `True`, `False`
- **Menos unario**: Analizado como `Binary { left: IntLiteral(0), op: Minus, right }` (sin `Unary::Neg` separado)
- **`not` / `!`**: `Unary { op: Not/Bang, right }`
- **Grupos `(`...`)`**: Expresión única → sin envolver; múltiples separadas por coma → `Tuple`
- **`[`...`]`**: Si el primer elemento va seguido de `::`, se analiza como `MapLiteral`; de lo contrario `ArrayLiteral`
- **`{`...`}`**: Analizado como `ArrayOrSetLiteral` (tipo resuelto en semántica o tiempo de compilación)
- **`set:N { }` etc.**: `SetLiteral` explícito con `SetType` conocido
- **`map { schema=[...] data=[...] }`**: `MapLiteral` explícito
- **`table { columns=[...] rows=[...] }`**: `TableLiteral`
- **`random.choice from expr`**: `RandomChoice`
- **`date(...)`**: `DateLiteral`
- **`net.get/post/put/delete/patch(...)` etc.**: `NetCall` o `NetRespond`
- **`<<<...>>>`**: `RawBlock`
- **`.terminal!cmd`**: `TerminalCommand`

Analizadores de infijo clave:
- **`.`**: `parse_dot_infix` — produce `MethodCall` si va seguido de `(`, de lo contrario `MemberAccess`; también gestiona acceso por índice `.[key]`
- **`[`**: `parse_index_infix` → `Index`
- **`->`**: `parse_lambda_infix` → `Lambda`
- **Todos los operadores binarios**: `Binary { left, op, right }`

## Post-procesamiento de `parse_expr_stmt()`

Tras analizar una sentencia de expresión completa, `parse_expr_stmt()` comprueba si el resultado es un `MethodCall`:
- Nombre de método `bind` con 2 args y segundo arg es `Identifier` → reescrito como `StmtKind::JsonBind`
- Nombre de método `inject` con 2 args → reescrito como `StmtKind::JsonInject`

Esto permite la sintaxis de azúcar `json.bind("path", target);` y `json.inject(mapping, table);` a nivel de sentencia.

## Expander (`src/parser/expander.rs`)

El Expander se ejecuta **después** del análisis, **antes** del análisis semántico. Es una pasada de reescritura de árbol independiente.

### Responsabilidades

**Resolución de includes**: `include "file.xcx";` se reemplaza por el AST en línea de ese archivo. Las dependencias circulares se detectan mediante `visiting_files: HashSet<PathBuf>`. Los archivos se deduplicán mediante `included_files: HashSet<PathBuf>` (cada archivo se incluye solo una vez a menos que tenga alias).

**Prefijación de alias**: `include "math.xcx" as math;` hace que todos los nombres de nivel superior de ese archivo sean renombrados a `math.nombre`. Los sitios de llamada (`math.sin(x)`) se reescriben de `MethodCall` a `FunctionCall { name: "math.sin" }` mediante `expand_expr_inplace`. Las funciones `prefix_program()` / `prefix_stmt_impl()` / `prefix_expr_impl()` recorren todo el sub-AST renombrando todas las referencias a símbolos de nivel superior.

**Prefijación de nombres de fiber**: Las referencias `FiberDecl::fiber_name` también se preifjan para que las instanciaciones de fibers renombrados se resuelvan correctamente tras la prefijación.

**Prefijación de YieldFrom**: Las expresiones `StmtKind::YieldFrom` se recorren para que las llamadas a constructores de fiber dentro de `yield from` también sean renombradas.

**Nombres protegidos** (nunca prefijados): `json`, `date`, `store`, `halt`, `terminal`, `net`, `env`, `crypto`, `EMPTY`, `math`, `random`, `i`, `f`, `s`, `b`, `from`, `main`.

**Orden de búsqueda de rutas de include**:
1. Relativa al directorio del archivo actual
2. En el directorio `lib/` (relativo al CWD, luego subiendo desde la ruta del ejecutable)

## Definiciones del AST (`src/parser/ast.rs`)

### `Expr` — Nodos de expresión

| Variante | Descripción |
|---|---|
| `IntLiteral(i64)` | Constante entera |
| `FloatLiteral(f64)` | Constante float |
| `StringLiteral(StringId)` | Cadena internada |
| `BoolLiteral(bool)` | `true` / `false` |
| `Identifier(StringId)` | Nombre de variable o función |
| `Binary { left, op, right }` | Operación binaria |
| `Unary { op, right }` | Operación unaria (`not`, `!`) |
| `FunctionCall { name, args }` | Llamada a función por nombre internado |
| `MethodCall { receiver, method, args }` | Llamada con punto sobre un valor |
| `MemberAccess { receiver, member }` | Acceso con punto sin llamada |
| `Index { receiver, index }` | Índice por corchete `a[i]` |
| `Lambda { params, return_type, body }` | Lambda de flecha `x -> expr` |
| `ArrayLiteral { elements }` | `[a, b, c]` explícito |
| `ArrayOrSetLiteral { elements }` | `{a, b, c}` ambiguo — resuelto más tarde |
| `SetLiteral { set_type, elements, range }` | Set tipado con rango opcional |
| `MapLiteral { key_type, value_type, elements }` | Literal de mapa |
| `TableLiteral { columns, rows }` | Literal de tabla |
| `DateLiteral { date_string, format }` | `date("2024-01-01")` |
| `Tuple(Vec<Expr>)` | Lista entre paréntesis separada por comas |
| `NetCall { method, url, body }` | Expresión de llamada HTTP |
| `NetRespond { status, body, headers }` | Expresión de respuesta HTTP |
| `RawBlock(StringId)` | Contenido raw `<<<...>>>` |
| `TerminalCommand(cmd, arg)` | `.terminal !cmd` |
| `RandomChoice { set }` | `random.choice from set` |

### `Stmt` — Nodos de sentencia

Variantes clave: `VarDecl`, `Assign`, `Print`, `Input`, `If`, `While`, `For`, `Break`, `Continue`, `FunctionDef`, `FiberDef`, `FiberDecl`, `Return`, `Yield`, `YieldFrom`, `YieldVoid`, `Include`, `Serve`, `NetRequestStmt`, `JsonBind`, `JsonInject`, `Halt`, `Wait`, `ExprStmt`, `FunctionCallStmt`.

### `Type` — Sistema de tipos

`Int`, `Float`, `String`, `Bool`, `Date`, `Json`, `Array(Box<Type>)`, `Set(SetType)`, `Map(Box<Type>, Box<Type>)`, `Table(Vec<ColumnDef>)`, `Fiber(Option<Box<Type>>)`, `Builtin(StringId)`, `Unknown`.

Variantes de `SetType`: `N` (Natural), `Z` (Entero), `Q` (Racional/Float), `S` (Cadena), `C` (Char/Cadena), `B` (Booleano).

### `ForIterType`

`Range` (numérico `start to end`), `Array`, `Set`, `Fiber` — establecido por el verificador de tipos y usado por el compilador para emitir el patrón de bucle correcto.

### `ColumnDef`

```rust
pub struct ColumnDef {
    pub name:    StringId,
    pub ty:      Type,
    pub is_auto: bool,    // las columnas @auto se auto-incrementan al insertar
}
```

## Internador de Cadenas

Todos los valores de cadena (identificadores, literales de cadena, nombres de métodos) se internan mediante `Interner` en `StringId (u32)`. El internador se crea en el parser y se pasa a través de todas las fases posteriores. Esto significa que el checker, el compilador y la VM usan IDs numéricos para las comparaciones de nombres en lugar de comparaciones de `String`.