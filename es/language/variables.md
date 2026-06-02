# XCX 3.1 Variables y Constantes

## Declaración de Variables

Las variables se declaran usando la sintaxis `tipo: nombre = valor;`.

```xcx
i: age = 25;
f: pi = 3.14159;
s: name = "XCX";
b: is_ready = true;
```

Las variables también pueden declararse sin valor inicial; en ese caso toman el valor por defecto de su tipo (por ejemplo, `0` para `i`, `""` para `s`).

## Reasignación

Una vez declarada, una variable puede reasignarse usando el operador `=`.

```xcx
i: count = 0;
count = count + 1;
```

## Constantes

Las constantes se declaran usando la palabra clave `const`. Deben inicializarse en el momento de la declaración y no pueden modificarse posteriormente.

```xcx
const i: MAX_CONNECTIONS = 1024;
const s: VERSION = "2.0.0";
```

## Sin Sombreado de Variables

XCX **no** admite el sombreado de variables. Declarar una variable con el mismo nombre en cualquier ámbito — incluyendo un bloque anidado — es un **error en tiempo de compilación** (`RedefinedVariable`).

```xcx
i: x = 10;
if (true) then;
    i: x = 20;   --- ERROR DE COMPILACIÓN: RedefinedVariable
end;
```

Si necesitas cambiar un valor dentro de un bloque, usa reasignación en lugar de redeclaración:

```xcx
i: x = 10;
if (true) then;
    x = 20;      --- OK: reasignación a variable existente
    >! x;        --- 20
end;
>! x;            --- 20 (la variable global fue modificada)
```