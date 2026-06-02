> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/language/syntax.md).

# XCX 3.1 Conceptos Básicos de Sintaxis

## Comentarios

```xcx
--- Comentario de una línea
i: price = 100;  --- comentario en línea

---
Bloque de comentarios
multilínea.
*---
```

## Identificadores

Los identificadores son **sensibles a mayúsculas/minúsculas** y deben coincidir con el patrón `[a-zA-Z][a-zA-Z0-9_]*`.

| Válido         | Inválido     | Razón                       |
|---------------|-------------|------------------------------|
| `userData`    | `1stUser`    | No puede comenzar con dígito    |
| `user_data`   | `user-data`  | El guión es un operador de resta    |
| `counter1`    | `user name`  | Los espacios no están permitidos       |

Las palabras clave (`if`, `func`, `fiber`, `i`, `f`, `s`, `b`, etc.) están reservadas.

## Terminales de Instrucción

Las sentencias terminan con punto y coma `;`. Esto se aplica a declaraciones de variables, llamadas de función, asignaciones y directivas.

```xcx
i: x = 10;
>! "Hello";
```

## Bloques

Los bloques se abren con una palabra clave (p. ej., `then`, `do`, `{`) y se cierran con `end;`.

```xcx
if (x > 0) then;
    >! "Positive";
end;
```
