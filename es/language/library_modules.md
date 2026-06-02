# XCX 3.1 Biblioteca Estándar y Módulos

## Módulos Integrados

### crypto

Utilidades de criptografía.

| Método                              | Retorna | Descripción                                    |
|-------------------------------------|---------|------------------------------------------------|
| `crypto.hash(data, "bcrypt")`       | `s`     | Hashea una contraseña usando bcrypt            |
| `crypto.hash(data, "argon2")`       | `s`     | Hashea una contraseña usando argon2 (recomendado) |
| `crypto.hash(data, "base64_encode")`| `s`     | Codifica datos binarios/cadena a Base64        |
| `crypto.hash(data, "base64_decode")`| `s`     | Decodifica una cadena Base64 a binario/cadena  |
| `crypto.verify(password, hash, algo)` | `b`   | Retorna `true` si la contraseña coincide con el hash |
| `crypto.token(length)`              | `s`     | Genera un token hex aleatorio de la longitud indicada |

```xcx
s: hash  = crypto.hash(password, "bcrypt");
s: hash2 = crypto.hash(password, "argon2");
b: valid = crypto.verify(password, hash2, "argon2");
s: token = crypto.token(32);
```

### store (E/S de Archivos)

Todas las rutas deben ser **relativas** a la raíz del proyecto. Las rutas absolutas o el recorrido de rutas (`..`) disparan `halt.fatal`.

| Método              | Firma                     | Retorna  | Descripción                                          |
|---------------------|---------------------------|----------|------------------------------------------------------|
| `store.write(p, c)` | `(s, s) → b`              | `b`      | Sobreescribe el archivo. Crea directorios si es necesario. |
| `store.read(p)`     | `(s) → s`                 | `s`      | Retorna el contenido del archivo; `halt.fatal` si no existe. |
| `store.append(p, c)`| `(s, s) → b`              | `b`      | Agrega contenido al archivo. Lo crea si no existe.   |
| `store.exists(p)`   | `(s) → b`                 | `b`      | Comprueba la existencia. Sin efectos secundarios.    |
| `store.delete(p)`   | `(s) → b`                 | `b`      | Elimina un archivo o directorio (recursivo).         |
| `store.list(p)`     | `(s) → array:s`           | `array:s`| Retorna la lista de archivos y carpetas.             |
| `store.isDir(p)`    | `(s) → b`                 | `b`      | `true` si la ruta es un directorio.                  |
| `store.size(p)`     | `(s) → i`                 | `i`      | Tamaño del archivo en bytes.                         |
| `store.mkdir(p)`    | `(s) → b`                 | `b`      | Crea un directorio (recursivo).                      |
| `store.glob(pat)`   | `(s) → array:s`           | `array:s`| Retorna los archivos que coinciden con el patrón glob.|
| `store.zip(s, t)`   | `(s, s) → b`              | `b`      | Comprime la fuente en el zip de destino.             |
| `store.unzip(z, d)` | `(s, s) → b`              | `b`      | Extrae un zip en el destino.                         |

```xcx
store.write("log.txt", "First line");
store.append("log.txt", "\nSecond line");
s: content = store.read("log.txt");
if (store.exists("lock.pid")) then;
    >! "Already running";
end;
```

### env

| Método          | Firma          | Retorna    | Descripción                                                        |
|-----------------|----------------|------------|--------------------------------------------------------------------|
| `env.get(name)` | `(s) → s`      | `s`        | Retorna el valor de la variable de entorno; `halt.error` si no existe |
| `env.args()`    | `() → array:s` | `array:s`  | Retorna los argumentos de línea de comandos pasados al programa    |

```xcx
s: db_url = env.get("DATABASE_URL");

array:s: args = env.args();
for arg in args do;
    >! arg;
end;
```

### random

| Método                            | Firma               | Retorna | Descripción                                                                       |
|-----------------------------------|---------------------|---------|-----------------------------------------------------------------------------------|
| `random.choice from col`          | `(set/array) → T`   | `T`     | Elige un elemento aleatorio del conjunto o array.                                 |
| `random.int(min, max @step num)`  | `(i, i, @i) → i`    | `i`     | Elige un entero aleatorio en el rango `[min, max]`. Paso por defecto: `1`.        |
| `random.float(min, max @step num)`| `(f, f, @f) → f`    | `f`     | Elige un flotante aleatorio en el rango `[min, max]`. Paso por defecto: `0.5`.    |


```xcx
set:N: pool {1,,10};
i: picked = random.choice from pool;

--- Works on arrays too
array:s: words {"hello", "world"};
s: w = random.choice from words;

--- Picks 1, 3, 5, 7, or 9
i: odd = random.int(1, 10 @step 2);

--- Picks 0.0, 0.5, 1.0, 1.5, or 2.0
f: weight = random.float(0.0, 2.0);

--- Picks 0.0, 0.25, 0.5, 0.75, 1.0
f: precision = random.float(0.0, 1.0 @step 0.25);
```

### date (módulo)

```xcx
date: now = date.now();
```

---

## Sistema de Módulos

### include

Fusiona el código de otro archivo en el espacio de nombres actual.

```xcx
include "utils.xcx";
include "math.xcx" as m;

m.PI;
m.sqrt(16.0);
```

Sin alias — todos los símbolos están disponibles directamente en el espacio de nombres actual. Con alias — todos los símbolos de nivel superior llevan el prefijo: `alias.symbol`.

Las dependencias cíclicas se detectan y rechazan en tiempo de compilación.