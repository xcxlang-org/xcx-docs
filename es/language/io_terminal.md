> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/language/io_terminal.md).

# XCX 3.1 — E/S y Comandos del Sistema

## Salida de Consola (`>!`)

El operador `>!` imprime valores en `stdout`, seguido de un salto de línea.

```xcx
>! "Hello";
>! 42;
>! "Path: " + path;
```

Las **secuencias de escape** como `\n` (nueva línea), `\t` (tabulación) y `\r` (retorno de carro) son compatibles.

## Entrada del Usuario (`>?`)

El operador `>?` lee una línea de `stdin` e intenta analizarla en la variable destino.

```xcx
i: age;
>! "Enter age:";
>? age;
```

## Retardo (`@wait`)

Pausa la ejecución de la VM durante un número específico de milisegundos.

```xcx
@wait 1000; --- Waits for 1 second
```

> `@wait` es una operación **bloqueante**. No cede el control de la fibra.

## Entrada sin Búfer (`input`)

El módulo `input` permite leer teclas individuales sin esperar a que se pulse Enter.

### Métodos

| Llamada              | Retorna | Descripción                                               |
|----------------------|---------|-----------------------------------------------------------|
| `input.key()`        | `s`     | Devuelve la tecla si está disponible, `""` si no lo está  |
| `input.key() @wait`  | `s`     | Espera una tecla y la devuelve                            |
| `input.ready()`      | `b`     | `true` si hay una tecla esperando en el búfer             |

### Constantes de Teclas

| Valor           | Tecla           |
|-----------------|-----------------|
| `"UP"`          | Flecha arriba   |
| `"DOWN"`        | Flecha abajo    |
| `"LEFT"`        | Flecha izquierda|
| `"RIGHT"`       | Flecha derecha  |
| `"ENTER"`       | Enter           |
| `"ESC"`         | Escape          |
| `"BACKSPACE"`   | Retroceso       |
| `"TAB"`         | Tab             |
| `"F1"` ... `"F12"` | F1–F12       |
| `"KEY_CTRL_C"`  | Ctrl+C          |
| `"KEY_CTRL_Z"`  | Ctrl+Z          |
| `"KEY_CTRL_S"`  | Ctrl+S          |

Los caracteres normales se devuelven directamente: `"a"`, `"Z"`, `"5"`, `" "`.

### Ejemplo

```xcx
s: k = input.key();
if (k == "UP") then;
    y = y - 1;
end;

--- esperar una tecla específica
s: confirm = input.key() @wait;
if (confirm == "q") then;
    return;
end;
```

## Comandos de Terminal (`.terminal`)

Interactúan directamente con el entorno del sistema o el proceso actual.

| Directiva               | Descripción                                         |
|-------------------------|-----------------------------------------------------|
| `.terminal !clear`      | Limpia la pantalla                                  |
| `.terminal !exit`       | Termina el proceso de la VM                         |
| `.terminal !run s`      | Ejecuta otro archivo XCX en un nuevo proceso        |
| `.terminal !raw`        | Modo raw — sin eco, sin búfer                       |
| `.terminal !normal`     | Restaura el modo de terminal normal                 |
| `.terminal !cursor on`  | Muestra el cursor                                   |
| `.terminal !cursor off` | Oculta el cursor                                    |
| `.terminal !move x y`   | Mueve el cursor a la posición `x`, `y` (`i`)        |
| `.terminal !write expr` | Imprime `expr` sin salto de línea al final          |

### Ejemplo — Bucle de Juego

```xcx
.terminal !raw;
.terminal !cursor off;
.terminal !clear;

while (true) do;
    s: k = input.key();
    if (k == "ESC") then; break; end;
    if (k == "UP") then; y = y - 1; end;
    .terminal !move x y;
    .terminal !write "@";
    @wait 16;
end;

.terminal !cursor on;
.terminal !normal;
```

## Manejo de Errores y Restricciones

- **`input.key()` en modo normal**: Devuelve `""` y muestra una advertencia.
- **`@wait` sobre `input.ready()`**: Error de compilación.
- **Directivas `.terminal`**: No son expresiones y no pueden asignarse a variables.
- **`.terminal !move`**: Los argumentos deben ser enteros (`i`).
- **Disponibilidad del Terminal**: La VM se detendrá con un error fatal si los manejadores de consola están redirigidos o no están disponibles.