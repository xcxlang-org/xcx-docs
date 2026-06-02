> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/language/string_methods.md).

# XCX 3.1 Métodos de Cadena

Los objetos de cadena en XCX son inmutables. Los métodos devuelven una **nueva cadena** y no modifican la original.

## Propiedades

- `.length`: Devuelve el número de puntos de código Unicode en la cadena (p. ej., `"zażółć".length` es 6). **Se usa sin paréntesis.**

## Secuencias de Escape

Los literales de cadena soportan secuencias de escape estándar:

| Secuencia | Efecto |
|----------|--------|
| `\n`     | Nueva línea |
| `\t`     | Tabulación horizontal |
| `\r`     | Retorno de carro |
| `\"`     | Comillas dobles |
| `\\`     | Barra invertida |
| `\xNN`   | Carácter hexadecimal (p. ej., `\x1b`) |
| `\NNN`   | Carácter octal (p. ej., `\033`) |

## Métodos

| Método               | Firma    | Descripción                                      |
|----------------------|--------------|--------------------------------------------------|
| `.upper()`           | `() → s`     | Convierte todos los caracteres a mayúsculas.            |
| `.lower()`           | `() → s`     | Convierte todos los caracteres a minúsculas.            |
| `.trim()`            | `() → s`     | Elimina espacios en blanco iniciales/finales.             |
| `.replace(f, t)`     | `(s, s) → s` | Reemplaza todas las ocurrencias de `f` con `t`.        |
| `.slice(s, e)`       | `(i, i) → s` | Devuelve subcadena desde índice `s` hasta `e`.      |
| `.indexOf(s)`        | `(s) → i`    | Devuelve índice de la primera ocurrencia, o -1.        |
| `.lastIndexOf(s)`    | `(s) → i`    | Devuelve índice de la última ocurrencia, o -1.         |
| `.startsWith(s)`     | `(s) → b`    | Devuelve verdadero si la cadena comienza con `s`.          |
| `.endsWith(s)`       | `(s) → b`    | Devuelve verdadero si la cadena termina con `s`.            |
| `.toInt()`           | `() → i`     | Analiza cadena a Entero; `halt.error` si falla.  |
| `.toFloat()`         | `() → f`     | Analiza cadena a Flotante; `halt.error` si falla.    |
| `.split(s)`          | `(s) → array:s` | Divide por separador `s`; devuelve arreglo de cadenas. |

## Examples

```xcx
s: raw = "  XCX-Language  ";
s: clean = raw.trim().lower().replace("-", "_"); --- "xcx_language"

i: start = "Programming".indexOf("gram");       --- 3
s: part = "Programming".slice(0, 4);            --- "Prog"

i: age = "25".toInt();
array:s: parts = "a,b,c".split(",");           --- {"a", "b", "c"}
```
