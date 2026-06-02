> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/compiler/language.md).

# Referencia del Lenguaje XCX — v3.1

> Referencia completa de la sintaxis y semántica del lenguaje XCX.

---

## Tabla de Contenidos

1. [Tipos de Datos](#tipos-de-datos)
2. [Variables y Constantes](#variables-y-constantes)
3. [Operadores](#operadores)
4. [Flujo de Control](#flujo-de-control)
5. [Funciones](#funciones)
6. [Fibers (Corrutinas)](#fibers-corrutinas)
7. [Colecciones](#colecciones)
8. [JSON](#json)
9. [Tablas](#tablas)
10. [Base de Datos (SQLite)](#base-de-datos-sqlite)
11. [Red (HTTP)](#red-http)
12. [Almacenamiento (Archivos)](#almacenamiento-archivos)
13. [Terminal](#terminal)
14. [Fecha y Hora](#fecha-y-hora)
15. [Aleatoriedad](#aleatoriedad)
16. [Criptografía](#criptografía)
17. [Diagnósticos y Halt](#diagnósticos-y-halt)
18. [Include](#include)

---

## Tipos de Datos

| Tipo XCX | Descripción | Ejemplo |
|---|---|---|
| `i` | Entero de 48 bits | `i: age = 25;` |
| `f` | Float de 64 bits | `f: pi = 3.14;` |
| `s` | Cadena UTF-8 | `s: name = "Alice";` |
| `b` | Booleano | `b: flag = true;` |
| `date` | Fecha/Hora (ms) | `date: d = date("2024-01-01");` |
| `json` | Cualquier JSON | `json: data = <<<{}>>>;` |
| `array:T` | Array tipado | `array:i nums = [1, 2, 3];` |
| `set:N/Q/Z/S/C/B` | Conjunto | `set:N mySet = set:N { 1,,100 };` |
| `map:K<->V` | Mapa | `map:s<->i scores = ["Alice" :: 100];` |
| `table` | Tabla relacional | `table: ...` |
| `fiber:T` | Corrutina | `fiber:i f = myFiber(10);` |
| `database` | Conexión SQLite | `database: db = database { ... };` |

### Casting de Tipos

```xcx
i(x)   --- a entero
f(x)   --- a float
s(x)   --- a cadena
b(x)   --- a booleano
```

---

## Variables y Constantes

```xcx
--- Declaración con tipo explícito
i: count = 0;
s: greeting = "Hello";

--- Constante (no puede reasignarse)
const f: PI = 3.14159;

--- Inferencia de tipos
var result = 42;        --- tipo inferido como int
var name = "Bob";       --- tipo inferido como string
```

### Asignación

```xcx
count = count + 1;
greeting = "World";
```

---

## Operadores

### Aritméticos

| Operador | Descripción | Ejemplo |
|---|---|---|
| `+` | Suma / Concatenación de cadenas | `a + b` |
| `-` | Resta | `a - b` |
| `*` | Multiplicación | `a * b` |
| `/` | División | `a / b` |
| `%` | Módulo | `a % b` |
| `^` | Exponenciación | `a ^ b` |
| `++` | Concatenación de dígitos | `12 ++ 34` = `1234` |

### Comparación

`==`, `!=`, `>`, `<`, `>=`, `<=`

### Lógicos

| Operador | Alternativa | Descripción |
|---|---|---|
| `AND` | `&&` | Y lógico |
| `OR` | `||` | O lógico |
| `NOT` | `!!` | No lógico |
| `HAS` | — | Contenencia (`"abc" HAS "b"`) |

### Operadores de Conjunto

| Operador | Símbolo | Unicode |
|---|---|---|
| `UNION` | | `∪` |
| `INTERSECTION` | | `∩` |
| `DIFFERENCE` | `\` | |
| `SYMMETRIC_DIFFERENCE` | | `⊕` |

### Operadores de Fecha

```xcx
date("2024-01-15") + 7    --- fecha + días
date("2024-01-15") - 7    --- fecha - días
date("2024-01-15") - date("2024-01-01")  --- diferencia en ms
```

---

## Flujo de Control

### If / ElseIf / Else

```xcx
if (condition) then;
    ...
elseif (other_condition) then;
    ...
else;
    ...
end;
```

### While

```xcx
while (condition) do;
    ...
end;
```

### For (Rango Numérico)

```xcx
for i in 1 to 100 do;
    >! i;
end;

--- Con paso
for i in 0 to 100 @step 5 do;
    >! i;
end;
```

### For (Array / Set / Fiber)

```xcx
for item in myArray do;
    >! item;
end;

for elem in mySet do;
    >! elem;
end;

for val in myFiber do;
    >! val;
end;
```

### Break y Continue

```xcx
while (true) do;
    if (condition) then;
        break;
    end;
    if (skip) then;
        continue;
    end;
end;
```

---

## Funciones

### Estilo de Llaves (similar a C)

```xcx
func add(i: a, i: b -> i) {
    return a + b;
}
```

### Estilo XCX (bloque con palabras clave)

```xcx
func:i: add(i: a, i: b) do;
    return a + b;
end;
```

### Lambdas

```xcx
var double = x -> x * 2;
var sum = (x, y) -> x + y;
```

### Llamada

```xcx
i: result = add(3, 4);
```

---

## Fibers (Corrutinas)

Los fibers son corrutinas cooperativas — no son hilos del SO.

### Definición de Fiber

```xcx
fiber counter(i: start) {
    i: n = start;
    while (true) do;
        yield n;
        n = n + 1;
    end;
}
```

### Instanciación de Fiber

```xcx
fiber:i: c = counter(1);
```

### Iteración sobre un Fiber

```xcx
for val in c do;
    >! val;
    if (val >= 10) then;
        break;
    end;
end;
```

### Reanudación Manual

```xcx
i: val = c.next();
b: done = c.isDone();
c.close();
```

### Delegación de Yield

```xcx
fiber pipeline(array:i data) {
    yield from processData(data);
}
```

### Fiber Void (sin valor)

```xcx
fiber doWork() {
    --- hace trabajo
    yield;  --- suspensión sin valor
}

fiber:void: w = doWork();
w.run();
```

---

## Colecciones

### Array

```xcx
array:i nums = [1, 2, 3, 4, 5];
nums.push(6);
i: first = nums.get(0);
nums.set(0, 99);
nums.delete(0);
i: size = nums.size();
b: has = nums.contains(3);
nums.sort();
nums.reverse();
s: joined = nums.join(", ");
```

### Set

```xcx
--- Conjunto natural con valores explícitos
set:N primes = set:N { 2, 3, 5, 7, 11 };

--- Conjunto con rango
set:Z mySet = set:Z { -10,,10 };

--- Conjunto con paso
set:N evens = set:N { 2,,100 @step 2 };

--- Operaciones
primes.add(13);
primes.remove(2);
b: hasFive = primes.contains(5);
i: count = primes.size();

--- Operaciones de conjuntos
set:N union = a UNION b;
set:N inter = a INTERSECTION b;
set:N diff  = a DIFFERENCE b;
set:N sym   = a SYMMETRIC_DIFFERENCE b;
```

### Map

```xcx
map:s<->i scores = ["Alice" :: 100, "Bob" :: 85];
scores.set("Charlie", 90);
i: aliceScore = scores.get("Alice");
scores.remove("Bob");
array:s keys = scores.keys();
```

---

## JSON

```xcx
--- Parseo de JSON
json: data = json.parse(jsonString);

--- Bloque raw (JSON en línea)
json: config = <<<{ "host": "localhost", "port": 8080 }>>>;

--- Acceso a campos
json: host = data.host;
json: nested = data.get("/server/host");

--- Modificación
data.set("/port", 9090);
data.push("/items", "newItem");

--- Binding a variables
s: hostname;
data.bind("/host", hostname);

--- Inyección en tabla
data.inject(mapping, myTable);

--- Comprobaciones
b: exists = data.exists("/optional");
i: count = data.size();
```

---

## Tablas

```xcx
--- Definición de tabla
table: users = table {
    columns = [
        id   :: i @auto,
        name :: s,
        age  :: i,
        email :: s
    ]
    rows = EMPTY
};

--- Inserción
users.insert("Alice", 30, "alice@example.com");
users.insert(name = "Bob", age = 25, email = "bob@example.com");

--- Consulta
table: adults = users.where(row -> row.age >= 18);
table: result = users.join(orders, "id", "userId");

--- Visualización
users.show();

--- Filas
i: count = users.count();
json: row = users.get(0);

--- Actualización
users.update(0, ["Alice Updated", 31, "alice@new.com"]);

--- Eliminación
users.delete(0);

--- Conversión a JSON
json: usersJson = users.toJson();

--- Guardado (@pk requerido para save())
table: products = table {
    columns = [id :: i @pk, name :: s, price :: f]
    rows = EMPTY
};
products.save("Laptop", 999.99);  --- insertar o actualizar
```

---

## Base de Datos (SQLite)

```xcx
--- Conexión
database: db = database {
    engine = "sqlite",
    path = "myapp.db",
    users = users,
    products = products
};

--- El esquema se sincroniza automáticamente

--- Inserción
db.insert(users, "Alice", 30, "alice@example.com");

--- Consulta
table: result = db.fetch(users);

--- Consulta filtrada
table: adults = db.fetch(users.where(row -> row.age >= 18));

--- SQL sin procesar
json: rows = db.queryRaw("SELECT * FROM users WHERE age > 25");

--- SQL parametrizado
json: res = db.exec("INSERT INTO logs (msg) VALUES (?)", ["event"]);

--- Transacciones
db.begin();
db.insert(users, "Charlie", 20, "charlie@example.com");
db.commit();

--- Eliminación (requiere .where())
db.remove(users).where(row -> row.age < 18);

--- Esquema
db.sync(users);
db.drop(users);
db.truncate(users);
```

---

## Red (HTTP)

### Peticiones HTTP Simples

```xcx
--- GET
json: resp = net.get("https://api.example.com/data");

--- POST
json: resp = net.post("https://api.example.com/users", payload);

--- PUT, DELETE, PATCH
json: resp = net.put("https://api.example.com/users/1", data);
json: resp = net.delete("https://api.example.com/users/1");
```

### Petición Completa

```xcx
net.request {
    method = "POST",
    url = "https://api.example.com/users",
    headers = ["Authorization" :: "Bearer token123"],
    body = payload,
    timeout = 5000
} as resp;

--- Comprobación de la respuesta
b: ok = resp.ok;
i: status = resp.status;
json: body = resp.body;
```

### Servidor HTTP

```xcx
fiber getHandler(json: req) {
    s: path = req.url;
    json: response = <<<{ "message": "Hello World" }>>>;
    net.respond(200, response);
}

serve: myServer {
    port = 8080,
    host = "0.0.0.0",
    workers = 4,
    routes = [
        ["GET /"    :: getHandler],
        ["POST /api" :: postHandler]
    ]
};
```

---

## Almacenamiento (Archivos)

```xcx
--- Lectura / Escritura
s: content = store.read("data.txt");
store.write("output.txt", "Hello World");
store.append("log.txt", "new line\n");

--- Comprobaciones
b: exists = store.exists("file.txt");
i: size = store.size("file.txt");
b: isDir = store.isDir("path/");

--- Operaciones de directorio
store.mkdir("newdir");
array:s files = store.list("mydir");
array:s matches = store.glob("*.xcx");

--- Eliminación
store.delete("temp.txt");

--- Archivado
b: zipped = store.zip("folder", "archive.zip");
b: ok = store.unzip("archive.zip", "output/");
```

---

## Terminal

```xcx
--- Imprimir sin salto de línea
.terminal!write "Hello ";
.terminal!write "World\n";

--- Limpiar pantalla
.terminal!clear;

--- Salir del programa
.terminal!exit;

--- Ejecutar comando del sistema
b: ok = .terminal!run "ls -la";

--- Modo raw (para apps interactivas)
.terminal!raw;

--- Leer tecla (no bloqueante)
s: key = input.key();

--- Leer tecla (bloqueante)
s: key = input.key() @wait;

--- Comprobar disponibilidad de entrada
b: ready = input.ready();

--- Cursor
.terminal!cursor on;
.terminal!cursor off;
.terminal!move 10 5;    --- columna 10, línea 5

--- Volver al modo normal
.terminal!normal;
```

---

## Fecha y Hora

```xcx
--- Fecha actual
date: now = date.now();

--- Literal de fecha
date: d = date("2024-01-15");
date: d2 = date("15/01/2024", "DD/MM/YYYY");

--- Componentes
i: year = d.year();
i: month = d.month();
i: day = d.day();
i: hour = d.hour();
i: minute = d.minute();
i: second = d.second();

--- Formato
s: formatted = d.format("YYYY-MM-DD");
s: withTime = d.format("YYYY-MM-DD HH:mm:ss");

--- Aritmética
date: tomorrow = d + 1;
date: yesterday = d - 1;
i: diff = date("2024-12-31") - date("2024-01-01");
```

---

## Aleatoriedad

```xcx
--- Entero aleatorio (rango)
i: n = random.int(1, 100);

--- Entero aleatorio con paso
i: even = random.int(2, 100 @step 2);

--- Float aleatorio
f: x = random.float(0.0, 1.0);

--- Float aleatorio con paso
f: y = random.float(0.0, 10.0 @step 0.5);

--- Elemento aleatorio de una colección
array:s colors = ["red", "green", "blue"];
s: picked = random.choice from colors;

set:N nums = set:N { 1,,10 };
i: val = random.choice from nums;
```

---

## Criptografía

```xcx
--- Hashing
s: hash = crypto.hash(password, "bcrypt");
s: hash2 = crypto.hash(password, "argon2");

--- Verificación
b: ok = crypto.verify(password, hash, "bcrypt");
b: ok2 = crypto.verify(password, hash2, "argon2");

--- Token aleatorio (hex)
s: token = crypto.token(32);   --- 32 caracteres hexadecimales

--- Base64
s: encoded = crypto.hash(data, "base64_encode");
s: decoded = crypto.hash(data, "base64_decode");
```

---

## Diagnósticos y Halt

```xcx
--- Imprimir (>!)
>! "Hello World";
>! age;
>! "Value: " + s(count);

--- Entrada (>?)
>? name;       --- lee en una variable existente
>? age;

--- Niveles de halt
halt.alert >! "Warning";          --- continúa la ejecución
halt.error >! "Logic Error";      --- detiene el frame
halt.fatal >! "Critical Error";   --- detiene el frame

--- Espera
@wait(1000);     --- esperar 1 segundo (ms)
@wait 500;       --- sintaxis alternativa
```

---

## Include

```xcx
--- Include simple (deduplicado — una sola vez)
include "utils.xcx";

--- Include con alias (todos los nombres reciben un prefijo)
include "math.xcx" as math;

--- Uso tras include con alias
f: result = math.sin(3.14);
f: cosVal = math.cos(0.0);
```

### Rutas de Búsqueda

1. Relativa al directorio del archivo actual
2. En el directorio `lib/` (CWD y rutas de biblioteca XCX)

---

## Variables de Entorno y CLI

```xcx
--- Variable de entorno
s: apiKey = env.get("API_KEY");

--- Argumentos de línea de comandos
array:s args = env.args();
s: firstArg = args.get(0);
```

---

## Comentarios

```xcx
--- Comentario de una línea

---
Comentario
multilínea
*---
```