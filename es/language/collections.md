> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/language/collections.md).

# XCX 3.1 Colecciones

## Arreglos

```xcx
array:i: nums {10, 20, 30};
nums.size();           --- 3
nums.get(0);           --- 10
nums.push(40);         --- adds 40 to the end
i: last = nums.pop();  --- removes and returns last element
nums.sort();           --- sorts in-place
nums.reverse();        --- reverses in-place
nums.show();           --- prints contents to terminal
```

### Métodos de Arreglo

| Método            | Firma        | Retorna | Descripción                                                         |
|-------------------|--------------|---------|---------------------------------------------------------------------|
| `.size()`         | `() → i`     | `i`     | Número de elementos                                                  |
| `.get(i)`         | `(i) → T`    | `T`     | Elemento en posición `i` (basado en 0); `halt.error` si fuera de rango  |
| `.push(val)`      | `(T) → b`    | `b`     | Añade elemento al final                                          |
| `.pop()`          | `() → T`     | `T`     | Elimina y devuelve el último elemento                                |
| `.insert(i, val)` | `(i, T) → b` | `b`     | Inserta en posición `i`, desplaza el resto; `halt.error` si fuera de rango |
| `.update(i, val)` | `(i, T) → b` | `b`     | Sobrescribe elemento en posición `i`; `halt.error` si fuera de rango   |
| `.delete(i)`      | `(i) → b`    | `b`     | Elimina elemento en posición `i`; `halt.error` si fuera de rango      |
| `.find(val)`      | `(T) → i`    | `i`     | Índice de la primera ocurrencia, o `-1`                                  |
| `.contains(val)`  | `(T) → b`    | `b`     | Verifica si el valor existe                                              |
| `.isEmpty()`      | `() → b`     | `b`     | `true` si está vacío                                                     |
| `.clear()`        | `() → b`     | `b`     | Elimina todos los elementos                                                |
| `.sort()`         | `() → b`     | `b`     | Ordena ascendente (en el lugar)                                          |
| `.reverse()`      | `() → b`     | `b`     | Invierte el orden (en el lugar)                                           |
| `.toStr()`        | `() → s`     | `s`     | Serializa arreglo a cadena con formato JSON                         |
| `.toJson()`       | `() → json`  | `json`  | Convierte arreglo a estructura JSON nativa                           |
| `.show()`         | `() → b`     | `b`     | Imprime contenidos en terminal                                         |

```xcx
array:i: nums {5, 2, 8, 1};
nums.sort();            --- {1, 2, 5, 8}
nums.reverse();         --- {8, 5, 2, 1}
nums.push(99);          --- {8, 5, 2, 1, 99}
i: last = nums.pop();   --- last = 99, nums = {8, 5, 2, 1}
nums.insert(1, 15);     --- inserts 15 at position 1
nums.update(0, 5);      --- sets element 0 to 5
nums.delete(3);         --- removes element at position 3
b: found = nums.contains(5);
i: idx   = nums.find(5);
b: empty = nums.isEmpty();
```

---

## Conjuntos

### Dominios

| Símbolo | Tipo              | Ejemplo                       |
|---------|-------------------|-------------------------------|
| `N`    | Natural (≥ 0)     | `set:N: s {0, 1, 2}`          |
| `Z`    | Entero           | `set:Z: s {-3, 0, 3}`         |
| `Q`    | Racional (Float)  | `set:Q: s {0.5, 1.0}`         |
| `S`    | Cadena            | `set:S: s {"a", "b"}`         |
| `B`    | Booleano           | `set:B: s {true, false}`      |
| `C`    | Carácter         | `set:C: s {"A",,"Z"}`         |

### Inicialización

Los conjuntos se pueden inicializar con valores explícitos o rangos. Los rangos son **inclusivos** en ambos lados.

```xcx
set:N: small  {1,,5};                  --- {1, 2, 3, 4, 5}
set:N: evens  {0,,100 @step 2};        --- {0, 2, 4, ...}
set:Q: thirds {0.0,,1.0 @step 0.33};
set:C: letters {"A",,"Z"};            --- all uppercase letters
```

Los conjuntos **eliminan automáticamente** elementos duplicados.

### Operaciones de Conjuntos

```xcx
set:N: setA {1,,5};
set:N: setB {3,,7};

set:N: u  = setA UNION setB;
set:N: i  = setA INTERSECTION setB;
set:N: d  = setA DIFFERENCE setB;
set:N: sd = setA SYMMETRIC_DIFFERENCE setB;

--- Los símbolos Unicode son equivalentes
setA ∪ setB
setA ∩ setB
setA \ setB
setA ⊕ setB
```

### Métodos de Conjuntos

| Método          | Firma | Retorna | Descripción                                |
|-----------------|-----------|---------|--------------------------------------------|
| `.size()`       | `() → i`  | `i`     | Número de elementos                         |
| `.isEmpty()`    | `() → b`  | `b`     | `true` si está vacío                            |
| `.contains(v)`  | `(T) → b` | `b`     | Verifica pertenencia                          |
| `.add(v)`       | `(T) → b` | `b`     | Añade elemento (ignora duplicado)           |
| `.remove(v)`    | `(T) → b` | `b`     | Elimina elemento (sin efecto si no existe)     |
| `.clear()`      | `() → b`  | `b`     | Elimina todos los elementos                       |
| `.show()`       | `() → b`  | `b`     | Imprime `{elem, elem, ...}` en terminal     |

### Selección Aleatoria e Iteración

```xcx
--- Selección aleatoria de un conjunto:
i: picked_set = random.choice from small;

--- Selección aleatoria de un arreglo:
array:i: nums {1, 2, 3, 4, 5};
i: picked_arr = random.choice from nums;
```

Seleccionar de un conjunto o arreglo vacío devuelve `false`.

### Iteración

```xcx
for p in small do;
    >! p;
end;
```

---

## Mapas

```xcx
map: ages {
    schema = [s <-> i]
    data = [ "alice" :: 30, "bob" :: 25 ]
};

--- Mapa Vacío
map: scores {
    schema = [s <-> i]   --- both separators are equivalent (<-> and <=>)
    data = [EMPTY]
};
```

### Métodos de Mapas

| Método           | Firma       | Retorna   | Descripción                               |
|------------------|-----------------|-----------|-------------------------------------------|
| `.size()`        | `() → i`        | `i`       | Número de pares clave-valor                 |
| `.get(key)`      | `(K) → V`       | `V`       | Devuelve valor; `halt.error` si falta la clave|
| `.contains(key)` | `(K) → b`       | `b`       | Verifica si la clave existe                      |
| `.insert(k, v)`  | `(K, V) → b`    | `b`       | Inserta o sobrescribe                     |
| `.remove(key)`   | `(K) → b`       | `b`       | Elimina par; `false` si falta la clave      |
| `.keys()`        | `() → array:K`  | `array:K` | Devuelve arreglo de claves                     |
| `.values()`      | `() → array:V`  | `array:V` | Devuelve arreglo de valores                   |
| `.clear()`       | `() → b`        | `b`       | Elimina todos los pares                         |
| `.toStr()`       | `() → s`        | `s`       | Serializa mapa a cadena con formato JSON |
| `.show()`        | `() → b`        | `b`       | Imprime contenidos del mapa en terminal           |
| `.toJson()`      | `() → json`     | `json`    | Serializa mapa a objeto JSON           |

Las claves del mapa se convierten a cadenas en el objeto JSON resultante.

### Serialización de Mapas (toJson)

#### Signature
`.toJson() → json`

#### Descripción
Serializa el mapa a un objeto JSON. Todas las claves se convierten a cadenas usando su representación `.toString()` para cumplir con los requisitos de las claves del objeto JSON.

```xcx
map: scores {
    schema = [s <-> i]
    data = [ "alice" :: 100, "bob" :: 85 ]
};
json: j = scores.toJson();
```

**JSON Output:**
```json
{
    "alice": 100,
    "bob": 85
}
```

#### Comportamiento para Casos Especiales
- **Mapa Vacío**: Devuelve un objeto vacío `{}`.
- **Estructuras Anidadas**: Si un mapa contiene arreglos, otros mapas o tablas, se serializan recursivamente a sus equivalentes JSON.
- **Conversión de Claves**: Todas las claves del mapa se convierten a cadenas en el objeto JSON resultante usando su representación `.toString()` estándar.

#### Mapeo de Tipos
| Tipo XCX | Tipo JSON |
|----------|-----------|
| `i`      | `number`  |
| `f`      | `number`  |
| `s`      | `string`  |
| `b`      | `boolean` |
| `date`   | `string` (format `"YYYY-MM-DD HH:mm:ss"`) |
| `json`   | (unchanged) |

Siempre usa `.contains()` antes de `.get()`:

```xcx
if (ages.contains("alice")) then;
    >! ages.get("alice");
end;
```

---

## Tablas

Estructuras de datos relacionales con columnas de auto-incremento opcionales.

```xcx
table: products {
    columns = [ id :: i @auto, name :: s, price :: f ]
    rows = [ ("Laptop", 2999.99), ("Phone", 1499.50) ]
};

--- Tabla Vacía
table: logs {
    columns = [ id :: i @auto, msg :: s ]
    rows = [EMPTY]
};
```

El modificador `@auto` en una columna `i` crea una ID auto-incrementada — se omite en `.insert()` y `.add()`.

> [!NOTE]
> Los atributos de columna adicionales (`@pk`, `@unique`, `@optional`, `@default(v)`, `@fk(t.col)`) se usan al conectar una tabla a una base de datos. Ver [Documentación de Base de Datos](database.md) para más detalles.

### Acceso a Filas

```xcx
products[0].name    --- "Laptop" (azúcar sintáctico para .get(0))
products[1].price   --- 1499.50
```

### Métodos de Tabla

| Método               | Firma               | Retorna | Descripción                                      |
|----------------------|-------------------------|---------|--------------------------------------------------|
| `.count()`           | `() → i`                | `i`     | Número de filas                                   |
| `.get(i)`            | `(i) → row`             | `row`   | Fila en índice `i`                                 |
| `.insert(vals...)`   | `(T...) → b`            | `b`     | Añade fila (omite columnas `@auto`)                 |
| `.add(vals...)`      | `(T...) → b`            | `b`     | Alias para `.insert()` — comportamiento idéntico       |
| `.update(i, vals)`   | `(i, [T...]) → b`       | `b`     | Reemplaza valores de fila; columnas `@auto` preservadas   |
| `.delete(i)`         | `(i) → b`               | `b`     | Elimina fila en índice `i`                         |
| `.where(pred)`       | `(expr) → table`        | `table` | Filtra — devuelve una nueva tabla                    |
| `.join(t, pred)`     | `(table, pred) → table` | `table` | Combinación interna con otra tabla                    |
| `.toJson()`          | `() → json`             | `json`  | Serializa todas las filas a arreglo JSON de objetos   |
| `.show()`            | `() → b`                | `b`     | Imprime tabla en formato ASCII                     |

### Argumentos Nombrados para `.add()` e `.insert()`

Cuando una tabla tiene atributos de columna de base de datos, los valores se pueden pasar por nombre de columna en lugar de posición. Los argumentos nombrados son **opcionales** — las llamadas posicionales siguen siendo completamente válidas.

```xcx
table: users {
    columns = [
        id    :: i @auto @pk,
        name  :: s @unique,
        age   :: i,
        phone :: s @optional,
        role  :: s @default("user")
    ]
    rows = [EMPTY]
};

--- Posicional (retrocompatible)
users.add("Alice", 25, "", "user");

--- Nombrado
users.add(name = "Alice", age = 25, phone = "", role = "user");

--- Mixto — argumentos posicionales deben venir primero
users.add("Alice", age = 25, role = "admin");
```

**Separación de espacios de nombres.** El lado izquierdo de `=` es siempre el nombre de la columna. El lado derecho es una expresión del ámbito local. Estos son dos espacios de nombres independientes — sin conflicto:

```xcx
s: name = "Alice";
users.add(name = name, age = 25);
--- "name" izquierdo  = columna users.name
--- "name" derecho = variable local
```

Reglas: los argumentos posicionales deben preceder a los nombrados; las columnas `@auto` nunca se pueden pasar; omitir una columna requerida (no-`@optional`, no-`@default`) es un error de compilación; nombres de columna duplicados en la misma llamada es un error de compilación. Ver [Documentación de Base de Datos](database.md) para la especificación completa.

### Filtrado (where)

```xcx
--- Sintaxis abreviada (nombres de columnas usables directamente)
table: expensive = products.where(price > 1000.0);
table: named     = products.where(name HAS "Pro");

--- Lambda
table: r = products.where(row -> row.price > 1000.0);

--- Encadenamiento
table: result = products
    .where(price > 1000.0)
    .where(name HAS "Pro");
```

> [!IMPORTANT]
> **Conflictos de Nombres en `.where()` (S301)**: Los nombres de columna tienen precedencia sobre las variables locales dentro de predicados. Si una variable local tiene el mismo nombre que una columna, renombra la variable para evitar un error de compilación.
>
> ```xcx
> --- Wrong (conflict: 'token' exists both as column and parameter)
> fiber verify(s: token) {
>     table: sess = db.sessions.where(token == token);
> };
>
> --- Correct
> fiber verify(s: t) {
>     table: sess = db.sessions.where(token == t);
> };
> ```

### Combinaciones

```xcx
--- Combinación basada en clave
table: report = users.join(orders, "id", "user_id");

--- Combinación Lambda
table: custom = tableA.join(tableB, (a, b) -> a.id == b.ref_id);
```

Cuando las tablas combinadas comparten un nombre de columna (que no sea la clave de combinación), la columna resultante se prefija con `{table_name}_`.

### Serialización (toJson)

#### Firma
`.toJson() → json`

#### Descripción
Serializa todas las filas de una tabla a un arreglo JSON, donde cada fila se convierte en un objeto con claves correspondientes a nombres de columnas. Las columnas `@auto` se incluyen en el resultado.

#### Formato
Siempre devuelve un arreglo JSON (`[...]`). Una tabla vacía devuelve `[]`.

```xcx
table: products {
    columns = [ id :: i @auto, name :: s, price :: f ]
    rows = [ ("Laptop", 2999.99), ("Phone", 1499.50) ]
};

json: result = products.toJson();
```

**JSON Output:**
```json
[
    {"id": 1, "name": "Laptop", "price": 2999.99},
    {"id": 2, "name": "Phone",  "price": 1499.50}
]
```

#### Mapeo de Tipos
| Tipo XCX        | Tipo JSON                                 |
|-----------------|-------------------------------------------|
| `i` / `int`     | `number`                                  |
| `f` / `float`   | `number`                                  |
| `s` / `str`     | `string`                                  |
| `b` / `bool`    | `boolean`                                 |
| `date`          | `string` (format `"YYYY-MM-DD HH:mm:ss"`) |

#### Comportamiento para Casos Especiales
- **Tabla Vacía**: Devuelve un arreglo vacío `[]`.
- **Tabla Filtrada**: Solo se serializan las filas actualmente en la tabla (después de `.where()`).
- **Tabla Combinada**: Se incluyen todas las columnas del resultado combinado.
