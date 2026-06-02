# XCX 3.1 Base de Datos

XCX 3.1 introduce soporte nativo para bases de datos relacionales mediante el tipo `database:` y un conjunto de métodos integrados. La versión 3.1 soporta únicamente **SQLite**.

---

## Tabla de Contenidos

1. [Declaración de Conexión (`database:`)](#1-declaración-de-conexión-database)
2. [Atributos de Columnas de Tabla](#2-atributos-de-columnas-de-tabla)
3. [Argumentos con Nombre](#3-argumentos-con-nombre)
4. [Captura de Resultado (`as`)](#4-captura-de-resultado-as)
5. [Limitaciones de Ámbito en Fibers (Windows)](#5-limitaciones-de-ámbito-en-fibers-windows)
6. [Referencia de API](#6-referencia-de-api)
   - 5.1 [DDL](#51-ddl)
   - 5.2 [Operaciones de Escritura](#52-operaciones-de-escritura)
   - 5.3 [Operaciones de Lectura](#53-operaciones-de-lectura)
   - 5.4 [Operaciones de Eliminación](#54-operaciones-de-eliminación)
   - 5.5 [Transacciones](#55-transacciones)
   - 5.6 [Otros](#56-otros)
7. [Mapeo de Tipos (XCX ↔ SQL)](#7-mapeo-de-tipos-xcx--sql)
8. [Restricciones de Seguridad](#8-restricciones-de-seguridad)
9. [Ejemplo Completo](#9-ejemplo-completo)

---

## 1. Declaración de Conexión (`database:`)

```xcx
database: app {
    engine  = "sqlite",
    path    = "app.db"
};
```

### Campos

| Campo      | Tipo | Requerido | Por defecto | Descripción                                           |
|------------|------|-----------|-------------|-------------------------------------------------------|
| `engine`   | `s`  | sí        | —           | Motor de base de datos. Actualmente solo `"sqlite"`.  |
| `path`     | `s`  | sí        | —           | Ruta al archivo `.db` (relativa a la raíz del proyecto).|
| `timeout`  | `i`  | no        | `5000`      | Tiempo límite de operación en ms.                     |
| `readonly` | `b`  | no        | `false`     | Modo de solo lectura.                                 |

### Comportamiento

- Las conexiones son **lazy** — se abren en el primer uso.
- Ante un fallo de conexión (por ejemplo, permisos insuficientes) se dispara `halt.error` automáticamente.
- SQLite crea el archivo de base de datos si no existe (salvo que `readonly = true`).
- Se permiten múltiples conexiones simultáneas — cada una tiene su propio nombre.

```xcx
database: main { engine = "sqlite", path = "main.db" };
database: logs { engine = "sqlite", path = "logs.db" };

yield main.sync(users);
yield logs.sync(events);
```

### Modelo de E/S

Todas las operaciones de BD que acceden al disco o al driver son operaciones de E/S:

| Contexto         | Comportamiento          |
|------------------|-------------------------|
| Dentro de un fiber | requiere `yield`      |
| Fuera de un fiber  | bloquea sincrónicamente |

Esto aplica a operaciones de **lectura** (`fetch`, `query`, `queryRaw`), **escritura** (`push`, `save`, `insert`, `exec`), **eliminación** (`remove`, `exec`) y **DDL** (`sync`, `drop`).

**Excepción — métodos sin `yield`.** Los métodos que solo operan sobre el estado de transacción o metadatos no requieren `yield` en ningún contexto:

- `db.has()`, `db.close()`, `db.isOpen()`
- `db.begin()`, `db.commit()`, `db.rollback()`

> `begin()`, `commit()` y `rollback()` no tocan datos — solo gestionan el estado de la transacción en el driver.

---

## 2. Atributos de Columnas de Tabla

Estos atributos extienden la declaración del bloque `table:` y son usados por el módulo de base de datos.

### Atributos

| Atributo        | Descripción                                                                  | Equivalente SQL     |
|-----------------|------------------------------------------------------------------------------|---------------------|
| `@pk`           | Clave primaria. Puede combinarse con `@auto`.                                | `PRIMARY KEY`       |
| `@unique`       | El valor debe ser único en la columna.                                       | `UNIQUE`            |
| `@optional`     | La columna puede ser `NULL` en SQL. XCX devuelve el valor por defecto del tipo al leer. | `NULL`     |
| `@default(v)`   | Valor por defecto de la columna en SQL.                                      | `DEFAULT v`         |
| `@fk(t.col)`    | Clave foránea que apunta a la columna `col` de la tabla `t`.                 | `REFERENCES t(col)` |

### Ejemplo

```xcx
table: users {
    columns = [
        id      :: i @auto @pk,
        name    :: s @unique,
        age     :: i,
        phone   :: s @optional,
        role    :: s @default("user")
    ]
    rows = [EMPTY]
};

table: posts {
    columns = [
        id      :: i @auto @pk,
        user_id :: i @fk(users.id),
        title   :: s,
        body    :: s @optional
    ]
    rows = [EMPTY]
};
```

### Comportamiento de `@optional`

XCX no tiene `null` como tipo independiente. Las columnas `@optional` en SQL pueden almacenar `NULL`, pero al leerlas en XCX el valor es reemplazado automáticamente por el valor por defecto del tipo:

| Tipo XCX | Valor cuando SQL es NULL |
|----------|--------------------------|
| `i`      | `0`                      |
| `f`      | `0.0`                    |
| `s`      | `""`                     |
| `b`      | `false`                  |

> **Nota:** `@optional` solo indica aceptabilidad de `NULL` en el lado SQL. XCX no preserva la distinción entre `NULL` y el valor por defecto — esa información se pierde en la lectura. Si la lógica de tu aplicación necesita distinguir estos casos, almacena un indicador en una columna separada.

### Reglas de Omisión de Columnas para `add()` e `insert()`

| Atributo de columna     | ¿Puede omitirse? | Qué va a SQL                   |
|-------------------------|------------------|--------------------------------|
| `@auto`                 | sí (siempre)     | valor autogenerado             |
| `@default(v)`           | sí               | valor `v` del `DEFAULT`        |
| `@optional`             | sí               | `NULL`                         |
| `@optional @default(v)` | sí               | valor `v` del `DEFAULT`        |
| sin atributos           | **no**           | error de compilación           |

> Las columnas `@auto` nunca pueden proporcionarse explícitamente — ni posicional ni por nombre. Intentarlo es un **error de compilación**.

---

## 3. Argumentos con Nombre

Los argumentos con nombre permiten pasar valores a `table.add()`, `table.insert()` y `db.insert()` **por nombre de columna** en lugar de por posición. Son una característica opcional y aditiva — todas las llamadas posicionales existentes siguen siendo válidas.

Los argumentos con nombre funcionan **únicamente** para operaciones de inserción en tablas. No aplican a funciones definidas por el usuario, fibers u otros métodos integrados. Los nombres de columna son conocidos estáticamente por el compilador desde el esquema `table:`.

### Sintaxis

```xcx
--- Posicional (compatible hacia atrás)
users.add("Alice", 25, "", "user");

--- Con nombre
users.add(name = "Alice", age = 25, phone = "", role = "user");

--- Con db.insert()
yield app.insert(users, name = "Alice", age = 25) as saved;
```

**Separación de espacios de nombres.** El lado izquierdo de `=` es siempre el nombre de la columna. El lado derecho es una expresión del ámbito local. Son dos espacios de nombres independientes — sin conflicto:

```xcx
s: name = "Alice";
users.add(name = name, age = 25);
--- "name" izquierdo  = columna users.name
--- "name" derecho = variable local
```

**Orden de evaluación.** Las expresiones del lado derecho se evalúan de izquierda a derecha en el orden escrito:

```xcx
users.add(name = get_name(), age = get_age());
--- get_name() se llama antes que get_age()
```

### Mezclar Posicional y con Nombre

Los argumentos posicionales y con nombre pueden mezclarse. **Los argumentos posicionales deben preceder a los argumentos con nombre.**

```xcx
--- OK — posicional antes que con nombre
users.add("Alice", age = 25, role = "admin");

--- ERROR DE COMPILACIÓN — con nombre antes que posicional
users.add(name = "Alice", 25, "admin");
```

**Regla de asignación.** Los argumentos posicionales se asignan a las columnas de izquierda a derecha en el orden de declaración, **saltando `@auto`**, hasta agotarse. Los argumentos con nombre rellenan las columnas restantes por nombre.

```xcx
--- "Alice" → name, el resto con nombre
users.add("Alice", age = 25, role = "admin");
--- phone omitido → NULL (@optional)

--- "Alice", 25 → name, age; role con nombre, phone omitido
users.add("Alice", 25, role = "mod");
```

Los argumentos posicionales se asignan secuencialmente **sin saltar**. Saltar una columna intermedia solo es posible mediante argumento con nombre o omitiéndola completamente (si es opcional):

```xcx
--- ERROR DE COMPILACIÓN — el compilador asigna: name="Alice", age="" — incompatibilidad de tipos
--- no hay forma de "saltar" age posicionalmente
users.add("Alice", "", "user");
```

Proporcionar la misma columna tanto posicionalmente como por nombre es un **error de compilación**:

```xcx
--- ERROR DE COMPILACIÓN — name proporcionado dos veces
users.add("Alice", name = "Bob", age = 25);
```

### Reglas en Tiempo de Compilación

| Regla | Comportamiento |
|-------|----------------|
| Se proporciona columna `@auto` | error de compilación |
| Nombre de columna desconocido | error de compilación |
| Nombre de columna duplicado | error de compilación |
| Columna requerida omitida | error de compilación |
| Argumento con nombre antes que posicional | error de compilación |

**Tabla de completitud:**

| Atributo de columna     | ¿Puede omitirse? | Valor cuando se omite                 |
|-------------------------|------------------|---------------------------------------|
| `@auto`                 | sí (siempre)     | valor autogenerado                    |
| `@default(v)`           | sí               | valor `v` del `DEFAULT`               |
| `@optional`             | sí               | `NULL` (→ valor por defecto en lectura)|
| `@optional @default(v)` | sí               | valor `v` del `DEFAULT`               |
| sin atributos           | **no**           | error de compilación                  |

```xcx
--- OK — role tiene @default("user"), phone es @optional
users.add(name = "Alice", age = 25);

--- ERROR DE COMPILACIÓN — age es requerido
users.add(name = "Alice");
```

---

## 4. Captura de Resultado (`as`)

`as` es un mecanismo general de XCX para capturar el resultado de una operación de bloque:

```xcx
--- Petición HTTP
net.request { method = "GET", url = "..." } as resp;

--- Operaciones de BD
yield app.insert(users, name = "Alice", age = 25) as saved;
yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
```

### Objeto de Resultado de Escritura/Eliminación

Las operaciones `db.insert()`, `db.save()`, `db.push()` y `db.exec()` devuelven un objeto con dos campos:

| Campo      | Tipo | Descripción                                                              |
|------------|------|--------------------------------------------------------------------------|
| `affected` | `i`  | Número de filas modificadas / insertadas por la operación                |
| `insertId` | `i`  | ID del último registro insertado (`@auto @pk`); `0` si no hubo inserción |

```xcx
yield app.insert(users, name = "Alice", age = 25) as saved;
>! "Nuevo ID de usuario: " + s(saved.insertId);

yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
>! "Eliminados: " + s(res.affected);
```

El uso de `as` es opcional — si el resultado no se necesita puede omitirse:

```xcx
yield app.save(users);
```

### Semántica de `affected` e `insertId`

| Operación                   | `affected`                       | `insertId`                          |
|-----------------------------|----------------------------------|-------------------------------------|
| `insert()` — fila única     | `1`                              | ID de la fila insertada             |
| `push()` — múltiples filas  | número de filas insertadas       | ID de la última fila insertada      |
| `save()` — upsert (inserción)| `1`                             | ID de la fila insertada             |
| `save()` — upsert (actualización)| `1`                         | `0`                                 |
| `exec(DELETE ...)`          | número de filas eliminadas       | `0`                                 |
| `exec(UPDATE ...)`          | número de filas actualizadas     | `0`                                 |

> Ante un fallo de operación (violación de restricción, sin conexión, `readonly`) se dispara `halt.error` y la ejecución no llega al código que usa el resultado `as`. El objeto de resultado siempre es válido si el código puede verlo.

---

## 5. Limitaciones de Ámbito en Fibers (Windows)

En XCX 3.1, específicamente en entornos Windows, inicializar una variable local de un fiber directamente con un resultado de base de datos puede ocasionalmente producir un error `Undefined variable` [S101] en las líneas siguientes.

### Solución Recomendada

Para garantizar la visibilidad completa dentro de bloques condicionales en un fiber, se recomienda **declarar primero el tipo de la variable** y luego asignar el resultado de la base de datos en una instrucción separada.

```xcx
fiber run(-> b) {
    --- Patrón recomendado
    b: exists_before;
    exists_before = db.has(logs);
    
    if (exists_before) then;
        >! "Table exists";
    end;
}
```

Este patrón garantiza que la variable esté correctamente registrada en la tabla de símbolos antes de ser accedida por el analizador. Esta limitación está prevista para una resolución arquitectónica nativa en XCX 4.0.

---

## 6. Referencia de API

### 5.1 DDL

```xcx
--- Crea la tabla SQL si no existe (basándose en el esquema de tabla XCX)
yield app.sync(users);

--- Elimina la tabla SQL
yield app.drop(users);

--- Comprueba si la tabla existe en la base de datos (sin tocar datos — sin yield)
b: exists = app.has(users);
```

### 5.2 Operaciones de Escritura

```xcx
--- INSERT de todas las filas de una tabla XCX local en SQL
yield app.push(users);

--- INSERT OR UPDATE (upsert) — requiere @pk; sin @pk = error de compilación
--- Implementación: INSERT INTO ... ON CONFLICT(pk) DO UPDATE SET ...
yield app.save(users);

--- INSERT de una sola fila — posicional o con nombre
yield app.insert(users, "Alice", 25) as saved;
yield app.insert(users, name = "Alice", age = 25) as saved;
```

Ante un error (violación de `@unique`, `@fk`, sin conexión) la operación dispara `halt.error`.

### 5.3 Operaciones de Lectura

```xcx
--- SELECT * — devuelve una tabla XCX nativa
table: all_users = yield app.fetch(users);

--- SELECT con filtro estilo XCX (compila a SQL WHERE)
table: adults = yield app.fetch(users).where(age > 18);

--- SELECT con SQL sin procesar — requiere el esquema como primer argumento
table: result = yield app.query(users, "SELECT * FROM users WHERE age > ?", [18]);

--- SQL sin procesar sin esquema — siempre devuelve un array JSON
json: raw = yield app.queryRaw("SELECT COUNT(*) as n FROM users");
--- raw = [{"n": 42}]

--- Obtener la primera fila
json: row = yield app.queryRaw("SELECT COUNT(*) as n FROM users").first();
--- row = {"n": 42}
--- halt.error si el resultado está vacío
```

#### `.first()` en el Resultado de `queryRaw`

`.first()` devuelve el primer elemento del array JSON como un objeto `json` individual. Dispara `halt.error` si la consulta no devolvió filas. El uso típico es para consultas de agregación (`COUNT`, `SUM`, `MAX`) y búsquedas que devuelven como máximo una fila.

```xcx
json: row = yield app.queryRaw("SELECT COUNT(*) as n FROM users").first();
i: total; row.bind("n", total);
>! "Usuarios: " + s(total);
```

Si no tienes la certeza de que el resultado será no vacío, comprueba el tamaño antes de usar `.first()`:

```xcx
json: results = yield app.queryRaw("SELECT * FROM users WHERE age > ?", [100]);
if (results.size() > 0) then;
    json: row = results.get(0);
end;
```

#### Operadores Soportados en `.where()`

El filtro `.where()` compila a una cláusula SQL `WHERE`. Se soporta un subconjunto limitado de expresiones XCX:

| Categoría     | Soportado                        |
|---------------|----------------------------------|
| Comparaciones | `==`, `!=`, `<`, `<=`, `>`, `>=` |
| Lógicos       | `AND`, `OR`, `NOT`               |
| Valores       | literales, variables locales     |
| Operandos izq.| solo nombres de columna          |

Las llamadas a funciones, métodos de cadena y expresiones anidadas no están soportados en `.where()` con `db.fetch()`. Usar una expresión no soportada es un **error de compilación**.

```xcx
--- OK
table: r = yield app.fetch(users).where(age > 18 AND role == "admin");

--- ERROR DE COMPILACIÓN — método de cadena no permitido en where() con fetch
table: r = yield app.fetch(users).where(name.lower() == "alice");
```

Encadenar `.where()` une las condiciones con `AND`:

```xcx
table: r = yield app.fetch(users)
    .where(age > 18)
    .where(role == "admin");
--- Equivalente SQL: WHERE age > 18 AND role = 'admin'
```

#### `db.fetch()` y las `rows` Locales

`db.fetch()` lee únicamente datos de la base de datos SQL — las `rows` locales declaradas en el bloque `table:` se ignoran. La definición de tabla local sirve solo como indicación de esquema.

```xcx
table: users {
    columns = [ id :: i @auto @pk, name :: s ]
    rows = [ ("Alice") ]   --- estas filas NO se usan en fetch
};

yield app.sync(users);

--- Devuelve solo las filas almacenadas en app.db
table: all = yield app.fetch(users);
```

Para insertar las `rows` locales en la base de datos, usa `db.push(users)` antes de `db.fetch()`.

### 5.4 Operaciones de Eliminación

```xcx
--- DELETE con filtro — .where() es obligatorio (D401)
yield app.remove(users).where(age < 18);

--- Eliminar todas las filas
yield app.truncate(users);

--- DELETE sin procesar con parámetros
yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
```

> **`db.remove()` sin `.where()` es un error de compilación (D401).** Para eliminar todas las filas, usa `db.truncate(users)` explícitamente.

### 5.5 Transacciones

```xcx
app.begin();
yield app.save(users);
yield app.save(posts);
app.commit();
```

> Si cualquier operación dentro de la transacción dispara `halt.error`, la transacción se **revierte automáticamente** antes de abortar el frame. La conexión vuelve al estado anterior a `begin()`.

`rollback()` sirve para cancelar explícitamente una transacción en lógica condicional antes de que ocurra un error:

```xcx
app.begin();
b: valid = validate_users(users);
if (NOT valid) then;
    app.rollback();
end;
yield app.save(users);
app.commit();
```

### 5.6 Otros

```xcx
app.close();
b: alive = app.isOpen();
```

---

## 7. Mapeo de Tipos (XCX ↔ SQL)

| Tipo XCX | Tipo SQL (SQLite)               |
|----------|---------------------------------|
| `i`      | `INTEGER`                       |
| `f`      | `REAL`                          |
| `s`      | `TEXT`                          |
| `b`      | `INTEGER` (0 / 1)               |
| `date`   | `TEXT` (`YYYY-MM-DD HH:mm:ss`)  |

---

## 8. Restricciones de Seguridad

| Restricción                               | Comportamiento                                                   |
|-------------------------------------------|------------------------------------------------------------------|
| Ruta que contiene `..`                    | `halt.fatal` — recorrido de ruta                                 |
| Ruta absoluta                             | `halt.fatal`                                                     |
| SQL sin procesar con entrada no validada  | El desarrollador es responsable de la sanitización — usa params `?` |
| `readonly = true` + operación DML         | `halt.error`                                                     |
| `save()` en tabla sin `@pk`               | error de compilación                                             |
| `remove()` sin `.where()`                 | error de compilación (D401)                                      |

Usa siempre sentencias preparadas con `?`:

```xcx
--- INCORRECTO
app.exec("DELETE FROM users WHERE name = '" + name + "'");

--- CORRECTO
app.exec("DELETE FROM users WHERE name = ?", [name]);
```

---

## 9. Ejemplo Completo

```xcx
database: app {
    engine = "sqlite",
    path   = "data.db"
};

table: users {
    columns = [
        id    :: i @auto @pk,
        name  :: s @unique,
        email :: s @unique,
        age   :: i,
        phone :: s @optional
    ]
    rows = [EMPTY]
};

yield app.sync(users);

fiber handle_get_users(json: req -> json) {
    table: all = yield app.fetch(users);
    yield net.respond(200, all.toJson());
};

fiber handle_create_user(json: req -> json) {
    json: body;
    req.bind("body", body);

    s: name;  body.bind("name", name);
    s: email; body.bind("email", email);
    i: age;   body.bind("age", age);

    --- Argumentos con nombre — legibles e independientes del orden
    yield app.insert(users, name = name, email = email, age = age) as saved;

    json: resp <<< {"ok": true, "id": 0} >>>;
    resp.set("id", saved.insertId);
    yield net.respond(201, resp);
};

serve: api {
    port   = 8080,
    routes = [
        "GET  /users" :: handle_get_users,
        "POST /users" :: handle_create_user
    ]
};
```