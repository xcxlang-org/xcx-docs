> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/language/functions_fibers.md).

# XCX 3.1 Funciones y Fibras

> [!IMPORTANT]
> **Regla de Alcance Global**: Las funciones y fibras deben ser declaradas al **nivel superior** de un archivo. No pueden estar anidadas dentro de bloques `if`/`while`/`for` u otras funciones/fibras de una manera que afecte el alcance — todos los nombres de función y fibra se registran globalmente durante la compilación.

## Funciones

### Estilos de Definición

Los parámetros siguen el patrón `type: name` (p. ej., `i: count`).

```xcx
--- Sin parámetros, sin valor de retorno
func hello() {
    >! "¡Hola!";
};

--- Sin parámetros, con valor de retorno
func get_pi(-> f) {
    return 3.14159;
};

--- Con parámetros, sin valor de retorno
func greet(s: who) {
    >! "¡Hola, " + who + "!";
};

--- Con parámetros y valor de retorno
func add(i: x, i: y -> i) {
    return x + y;
};
```

### Llamando Funciones

```xcx
hello();
f: pi = get_pi();
greet("Alice");
i: result = add(3, 4);
```

### Recursión

Profundidad máxima: **800 marcos**. Exceder esto dispara `halt.error`.

```xcx
func factorial(i: n -> i) {
    if (n <= 1) then;
        return 1;
    else;
        return n * factorial(n - 1);
    end;
};
>! factorial(5);   --- 120
```

---

## Fibras (Corrutinas)

Las fibras pueden suspender la ejecución con `yield` y reanudarse. Hay dos variantes: **tipificada** (yield devuelve un valor) y **vacía** (yield sin valor).

### Definición

```xcx
--- Fibra Tipificada: cada yield devuelve un valor de tipo T
fiber generator(i: max -> i) {
    for i in 1 to max do;
        yield i;
    end;
    return 0;
};

--- Fibra Vacía: yield sin un valor
fiber logger(s: msg) {
    if (msg == "") then; return; end;
    >! msg;
    yield;
};
```

### Instanciación

```xcx
fiber:i: gen  = generator(10);   --- tipificada
fiber:  log   = logger("init");  --- vacía
```

### Métodos de Fibra

| Método      | Funciona en   | Retorna | Descripción                              |
|-------------|------------|---------|------------------------------------------|
| `.next()`   | `fiber:T`  | `T`     | Reanuda; devuelve el próximo valor generado      |
| `.run()`    | `fiber:`   | `b`     | Reanuda la fibra vacía un paso              |
| `.isDone()` | ambos       | `b`     | `true` cuando la fibra ha terminado           |
| `.close()`  | ambos       | `b`     | Termina la fibra forzosamente          |

Llamar `.next()` cuando `isDone() == true` dispara `halt.alert` (R306).

### yield

```xcx
--- In a typed fiber
yield true;
yield score + 10;

--- In a void fiber
yield;
```

### return in Fibers

1. **Typed fiber** (`fiber ... -> T`): Requires `return expr;` with a value of type `T`. A bare `return;` without a value is a compile error (S210).
2. **Void fiber**: Uses `return;` to exit early.

```xcx
fiber:i: gen(i: max -> i) {
    for i in 1 to max do;
        if (i == 3) then;
            return 99;   --- Valid: terminates and returns 99
        end;
        yield i;
    end;
    return 0;            --- Required at end of typed fiber
};

fiber logger(s: msg) {
    if (msg == "") then;
        return;          --- Valid for void fiber
    end;
    >! msg;
};
```

### for Loop over a Fiber

```xcx
fiber:i: f = generator(5);
for val in f do;
    >! val;
end;
```

Equivalent to `while (!f.isDone()) { val = f.next(); ... }`. `break` inside the loop automatically calls `.close()`.

### Fiber as Function Parameter

```xcx
func run_checks(fiber:b: f) {
    while (f.isDone() == false) do;
        b: result = f.next();
        if (result == false) then;
            halt.alert >! "Step failed";
            return;
        end;
    end;
};

fiber:b: checks = login_pipeline(user, pass);
run_checks(checks);
```

### yield from (Delegation)

Delega la ejecución a una sub-fibra. El compilador transforma esto en un bucle que produce cada valor de la sub-fibra.

```xcx
fiber count(i: n -> i) {
    i: x = 0;
    while (x < n) do;
        yield x;
        x = x + 1;
    end;
    return 0;
};

fiber delegator(-> i) {
    fiber:i: sub = count(3);
    yield from sub;   --- yields: 0, 1, 2
    yield 100;        --- then yields: 100
    return 0;
};
```

### Manual Delegation

```xcx
fiber pipeline(s: user, s: pass -> b) {
    fiber:b: step1 = check_user(user);
    yield step1.next();

    fiber:b: step2 = check_pass(user, pass);
    yield step2.next();

    return false;
};
```
