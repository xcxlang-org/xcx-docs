# XCX 3.1 Operadores

## Operadores Aritméticos

| Operador | Descripción                          | Ejemplo    | Resultado |
|----------|--------------------------------------|------------|-----------|
| `+`      | Suma                                 | `10 + 3`   | `13`      |
| `-`      | Resta / Negación unaria              | `10 - 3`   | `7`       |
| `*`      | Multiplicación                       | `10 * 3`   | `30`      |
| `/`      | División (trunca para `i`)           | `10 / 3`   | `3`       |
| `%`      | Módulo (Resto)                       | `10 % 3`   | `1`       |
| `^`      | Exponenciación                       | `2 ^ 10`   | `1024`    |
| `++`     | Concatenación de dígitos enteros     | `48 ++ 77` | `4877`    |

## Operadores de Cadenas

| Operador | Descripción              | Ejemplo                  | Resultado        |
|----------|--------------------------|--------------------------|------------------|
| `+`      | Concatenación            | `"Hi " + "User"`         | `"Hi User"`      |
| `HAS`    | Búsqueda de subcadena    | `"Search" HAS "ear"`     | `true`           |

> [!TIP]
> La concatenación de cadenas convierte automáticamente números y booleanos a cadenas (por ejemplo, `"Value: " + 42` → `"Value: 42"`).

## Operadores Lógicos

| Operador | Alias | Descripción   |
|----------|-------|---------------|
| `AND`    | `&&`  | AND lógico    |
| `OR`     | `||`  | OR lógico     |
| `NOT`    | `!!`  | NOT lógico    |

## Operadores de Comparación

Se usan para comparar valores del mismo tipo.

| Operador | Descripción               |
|----------|---------------------------|
| `==`     | Igual a                   |
| `!=`     | Distinto de               |
| `<`      | Menor que                 |
| `>`      | Mayor que                 |
| `<=`     | Menor o igual que         |
| `>=`     | Mayor o igual que         |

## Operadores de Conjuntos

| Palabra                | Símbolo | Descripción                  |
|------------------------|---------|------------------------------|
| `UNION`                | `∪`     | Unión de conjuntos           |
| `INTERSECTION`         | `∩`     | Intersección de conjuntos    |
| `DIFFERENCE`           | `\`     | Diferencia de conjuntos      |
| `SYMMETRIC_DIFFERENCE` | `⊕`     | Diferencia simétrica         |

## Precedencia de Operadores (de mayor a menor)

1. Llamadas a funciones, métodos, indexación `[]`
2. Negación unaria `-`, `!!` (NOT)
3. Exponenciación `^`
4. Multiplicación `*`, División `/`, Módulo `%`
5. Operaciones de conjuntos `UNION`, `INTERSECTION`, `DIFFERENCE`, `SYM_DIFF`
6. Suma `+`, Resta `-`, Concatenación `++`
7. Comparaciones `HAS`, `>`, `<`, `>=`, `<=`
8. Igualdad `==`, `!=`
9. Lógico `AND`, `&&`
10. Lógico `OR`, `||`