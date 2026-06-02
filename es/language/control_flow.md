> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/language/control_flow.md).

# XCX 3.1 Flujo de Control

## Sentencias Condicionales (if/elseif/else)

```xcx
if (condition) then;
    --- block
elseif (other_condition) then;
    --- block
else;
    --- block
end;
```

**Alias:**

| Palabra clave   | Aliases        |
|-----------|----------------|
| `elseif`  | `elif`, `elf`  |
| `else`    | `els`          |

Todos los formularios son equivalentes y pueden mezclarse:

```xcx
i: score = 75;
if (score >= 90) then;
    >! "A";
elif (score >= 75) then;
    >! "B";
elf (score >= 60) then;
    >! "C";
els;
    >! "F";
end;
```

---

## Bucles

### For de Rango

El rango es **inclusivo** en ambos lados.

```xcx
for i in 1 to 3 do;
    >! i;
end;
--- imprime: 1, 2, 3
```

### For con Paso (`@step`)

```xcx
for j in 0 to 6 @step 2 do;
    >! j;
end;
--- imprime: 0, 2, 4, 6
```

### Iteración de Colecciones

Funciona en arreglos, conjuntos y fibras. La variable del bucle recibe el **elemento**, no el índice.

```xcx
--- Arreglo
array:i: nums {10, 20, 30};
for el in nums do;
    >! el;
end;

--- Conjunto
set:N: primes {2, 3, 5, 7, 11};
for p in primes do;
    >! p;
end;

--- Fibra
fiber:i: f = gen(5);
for val in f do;
    >! val;
end;
```

### Bucle While

```xcx
i: cnt = 0;
while (cnt < 3) do;
    cnt = cnt + 1;
    >! cnt;
end;
```

### Break y Continue

`break` sale del bucle actual. `continue` salta a la siguiente iteración. Ambos afectan solo el **bucle inmediatamente envolvente**.

```xcx
for n in 1 to 5 do;
    if (n % 2 == 0) then; continue; end;
    if (n == 5) then; break; end;
    >! n;
end;
```

> [!NOTE]
> `break` dentro de un bucle `for` sobre una fibra automáticamente llama `.close()` en esa fibra.
