> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/compiler/lexer.md).

# Escáner Léxico de XCX — v3.1

El Escáner Léxico es responsable de convertir el flujo de bytes brutos del código fuente en una secuencia de tokens discretos.

## Detalles de Implementación

- **Archivo**: `src/lexer/scanner.rs`
- **Técnica**: Análisis manual, ansioso, byte por byte en `&[u8]`.
- **API**: Un único método `next_token(&mut self, interner: &mut Interner) -> Token`, llamado bajo demanda por el analizador (no un iterador).
- **Anticipación**: Anticipación de un byte via `peek()`, dos bytes via `peek_next()` / `peek_at(offset)`.

## Estructura Interna

```rust
pub struct Scanner<'a> {
    source:   &'a [u8],   // referencia prestada de la fuente original — sin asignación
    pos:      usize,      // posición de byte
    char_pos: usize,      // posición de carácter Unicode (para Span.col)
    line:     usize,
    col:      usize,
}
```

El `Scanner` toma prestada la cadena de fuente como un segmento de bytes. No hay conversión previa a `Vec<char>` — todo el análisis es a nivel de byte. `char_pos` se incrementa solo para bytes que no son bytes de continuación UTF-8 (`10xxxxxx`), por lo que los recuentos de caracteres Unicode permanecen correctos para la notificación de Span sin decodificar cada carácter.

## Tipos de Tokens

Los tokens se definen en `src/lexer/token.rs` como la enumeración `TokenKind`. Cada `Token` lleva un `Span { line, col, len }` para notificación de errores. `len` se mide en caracteres Unicode (via deltas de `char_pos`), no en bytes.

Categorías clave:

| Categoría | Ejemplos |
|---|---|
| Literales | `IntLiteral(i64)`, `FloatLiteral(f64)`, `StringLiteral(StringId)`, `True`, `False` |
| Palabras clave de tipo | `TypeI`, `TypeF`, `TypeS`, `TypeB`, `Array`, `Set`, `Map`, `Table`, `Json`, `Date`, `Fiber` |
| Palabras clave de tipo de conjunto | `TypeSetN`, `TypeSetQ`, `TypeSetZ`, `TypeSetS`, `TypeSetB`, `TypeSetC` |
| Control de flujo | `If`, `Then`, `ElseIf`, `Else`, `End`, `While`, `Do`, `For`, `In`, `To`, `Break`, `Continue` |
| Funciones/Fibras | `Func`, `Return`, `Fiber`, `Yield` |
| Operadores | `Plus`, `Minus`, `Star`, `Slash`, `Caret`, `PlusPlus`, `Has`, `And`, `Or`, `Not` |
| Operadores de conjunto | `Union`, `Intersection`, `Difference`, `SymDifference` |
| Puntuación especial | `GreaterBang` (`>!`), `GreaterQuestion` (`>?`), `DoubleColon` (`::`), `DoubleComma` (`,,`), `Bridge` (`<->`) |
| Funciones integradas | `Net`, `Serve`, `Store`, `Halt`, `Terminal`, `Json`, `Date` |
| Especial | `RawBlock(StringId)`, `AtStep`, `AtAuto`, `AtWait` |

## Características Especiales de Análisis

### Bloques Crudos
Delimitados por `<<<` y `>>>`. Todo entre ellos se captura como un único token `RawBlock(StringId)`, utilizado para JSON en línea o datos de cadena multilínea.

```
<<<
  { "key": "value" }
>>>
```

La detección usa `self.source[self.pos..].starts_with(b"<<<")` — abierto como parte de la desambiguación de `<` en el brazo de coincidencia principal. El contenido crudo se captura byte por byte hasta que se encuentra `>>>`.

### Comentarios
XCX usa `---` como delimitador de comentario. La detección usa `self.source[self.pos..].starts_with(b"---")`:
- **Una línea**: `--- this is a comment` (contenido no en blanco en la misma línea después de `---`)
- **Múltiples líneas**: `---` seguido solo de espacios en blanco hasta el final de línea abre un bloque, cerrado por `*---`

El cierre multilínea se detecta con `self.source[self.pos..].starts_with(b"*---")`. El escáner anticipa después de consumir `---` para decidir qué modo se aplica.

### Operadores de Conjunto Unicode
El escáner reconoce símbolos Unicode via `starts_with` en sus secuencias de bytes UTF-8:
- `∪` → `TokenKind::Union`
- `∩` → `TokenKind::Intersection`
- `\` (barra invertida ASCII) → `TokenKind::Difference`
- `⊕` → `TokenKind::SymDifference`

Para operadores Unicode de múltiples bytes, `advance()` se llama el número apropiado de veces adicionales para consumir los bytes de continuación restantes después del byte principal (`c >= 128`).

### Desambiguación Else/ElseIf
El escáner anticipa después de reconocer `else` / `els` para verificar si la siguiente palabra es `if` — si es así, colapsa las dos palabras en un único token `ElseIf`. La posición guardada (`after_ws_pos`, `after_ws_char_pos`, `after_ws_line`, `after_ws_col`) permite que el escáner retroceda si la siguiente palabra no es `if`.

### Directivas `@`
Los tokens que comienzan con `@` se analizan consumiendo bytes alfabéticos ASCII y coincidiendo el resultado:
- `@step` → `AtStep`
- `@auto` → `AtAuto`
- `@wait` → `AtWait`

Las secuencias `@` desconocidas producen `TokenKind::Unknown('@')`.

### Punto-Punto (`..`) → `To`
Dos puntos consecutivos `..` se analizan como el token `To` (usado en expresiones de rango), distinto de un único `.` (`Dot`).

### Coma Doble (`,,`) → `DoubleComma`
Dos comas consecutivas `,,` se analizan como `DoubleComma` (usado en literales de rango de conjunto: `set:N { 1,,10 }`).

### Análisis de Identificador
`identifier()` captura una ejecución contigua de bytes alfanuméricos ASCII, guiones bajos, y bytes `>= 128` (para incluir identificadores UTF-8). El rango de bytes capturado se convierte via `std::str::from_utf8` y se convierte a minúsculas para la coincidencia de palabras clave. Las coincidencias sensibles a mayúsculas se verifican primero (por ejemplo, `"N"`, `"Q"`, `"Z"` para tipos de conjunto, `"UNION"`, `"HAS"`, `"AND"` para variantes de palabras clave en mayúsculas). Los identificadores que no son palabras clave se pasan a `Interner::intern()`.

### Análisis de Números
`number()` acumula bytes de dígitos ASCII. Si se encuentra un `.` seguido de un dígito, el token se convierte en `FloatLiteral`. El rango de bytes se convierte via `str::from_utf8` y se analiza con `.parse()`.

### Análisis de Cadenas
`string()` procesa secuencias de escape (`\n`, `\t`, `\r`, `\"`, `\\`, octal `\NNN`, hexadecimal `\xHH`) byte por byte, construyendo un `Vec<u8>` que luego se convierte via `String::from_utf8`. El resultado se interna.

## Asignación de Cadenas

Todos los identificadores y literales de cadena se pasan a través de `Interner::intern()`, devolviendo un `StringId (u32)`. La `String` bruta se almacena una vez en el `Vec<String>` interno del internador; el resto de la canalización funciona con IDs numéricos, eliminando comparaciones de montón durante la verificación de tipos y la compilación.