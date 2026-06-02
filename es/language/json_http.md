> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/language/json_http.md).

# XCX 3.1 — JSON y HTTP

## JSON

Los objetos JSON en XCX 3.1 son **mutables**.

### Creación

```xcx
json: config <<< {"port": 8080, "debug": false} >>>;
json: user   <<< {"name": "", "age": 0} >>>;
```

Los valores en el literal pueden ser marcadores de posición (`""`, `0`, `false`) para rellenarse después mediante `.set()`.

### Serialización desde Colecciones

También puedes crear objetos y arrays JSON directamente desde colecciones XCX usando el método `.toJson()`. Está disponible para:
- **Maps**: Devuelve un objeto JSON.
- **Tables**: Devuelve un array JSON de objetos.

Consulta la [Documentación de Colecciones](collections.md) para más detalles sobre el mapeo y el comportamiento.

### Parseo

```xcx
json: parsed = json.parse(raw_string);
```

> [!CAUTION]
> **Pánico en JSON inválido (R305)**: Si el parseo falla, la VM se termina inmediatamente. Verifica el contenido del string antes de parsearlo.

### Patrón de Mutabilidad

Declara un esquema con valores cero y luego rellénalo:

```xcx
json: resp <<< {"token": "", "role": "", "uid": 0, "ok": false} >>>;
resp.set("token", crypto.token(32));
resp.set("role",  "admin");
resp.set("uid",   42);
resp.set("ok",    true);
yield net.respond(200, resp);
```

### Métodos JSON

| Método                    | Firma                 | Retorna | Descripción                                                         |
|---------------------------|-----------------------|---------|---------------------------------------------------------------------|
| `.exists(path)`           | `(s) → b`             | `b`     | Comprueba si la ruta existe y no es nula                            |
| `.get(path/idx)`          | `(s/i) → json`        | `json`  | Obtiene el elemento en la ruta o índice                             |
| `.bind(path, var)`        | `(s, ref) → b`        | `b`     | Extrae el valor en una variable XCX **previamente declarada**       |
| `.set(path, val)`         | `(s, T) → b`          | `b`     | Establece el valor en la ruta; crea la clave si no existe           |
| `.push(val)`              | `(json) → b`          | `b`     | Agrega un elemento a un nodo array JSON                             |
| `.size()` / `.count()`    | `() → i`              | `i`     | Número de claves (objeto) o elementos (array)                       |
| `.toStr()`                | `() → s`              | `s`     | Serializa a string JSON                                             |
| `.inject(path, map, tbl)` | `(s, map, table) → b` | `b`     | Importación masiva de un array JSON en una tabla XCX                |
| `.first()`                | `() → json`           | `json`  | Devuelve el primer elemento de un array JSON; halt.error si está vacío |

> [!NOTE]
> **`.push()` sobre arrays JSON**: `.push()` funciona exclusivamente en nodos JSON que son arrays (`[]`). Llamar a `.push()` sobre un objeto JSON (`{}`) resulta en un `halt.error`.
>
> ```xcx
> json: data <<< {"items": []} >>>;
> json: obj <<< {"id": 1} >>>;
> data.get("items").push(obj);   --- OK: "items" es un array
> data.push(obj);                --- halt.error: data es un objeto, no un array
> ```

> [!IMPORTANT]
> **Sintaxis de `.bind()`**: El segundo argumento debe ser una **variable previamente declarada**. No puedes declarar el tipo en línea.
>
> ```xcx
> --- Incorrecto
> req.bind("ip", s: ip);
>
> --- Correcto
> s: ip;
> req.bind("ip", ip);
> ```

### Notación de Rutas

Se admite tanto la notación de punto como la de corchetes para el acceso anidado:

```xcx
--- Campo anidado
json: cfg <<< {"server": {"host": "localhost"}} >>>;
s: host;
cfg.bind("server.host", host);

--- Índice de array
json: resp <<< {"items": []} >>>;
resp.set("items[0]", first_item);
resp.set("items[1]", second_item);
```

### `.inject()` — Importación Masiva

Importa un array JSON directamente en una tabla XCX:

```xcx
json: data <<< {"users":[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]} >>>;
table: imported { columns=[uid::i, uname::s] rows=[EMPTY] };
map: mapping { schema=[s<->s] data=["uid"::"id", "uname"::"name"] };
data.inject("users", mapping, imported);
imported.show();
```

### JSON en Peticiones HTTP

Dentro de los manejadores HTTP, `json: req` tiene esta estructura:

```json
{
    "method":  "POST",
    "path":    "/api/login",
    "query":   { "page": "1" },
    "headers": { "authorization": "Bearer ..." },
    "body":    { ... },
    "ip":      "1.2.3.4"
}
```

```xcx
s: ip;
req.bind("ip", ip);

json: body;
req.bind("body", body);

json: headers;
req.bind("headers", headers);

s: auth;
headers.bind("authorization", auth);
```

---

## Cliente HTTP

### API de Alto Nivel

| Método                  | Firma               | Retorna |
|-------------------------|---------------------|---------|
| `net.get(url)`          | `(s) → json`        | `json`  |
| `net.post(url, body)`   | `(s, json) → json`  | `json`  |
| `net.put(url, body)`    | `(s, json) → json`  | `json`  |
| `net.delete(url)`       | `(s) → json`        | `json`  |

```xcx
json: resp = net.get("https://api.example.com/users");

json: body <<< {"name": "Alice"} >>>;
json: resp = net.post("https://api.example.com/users", body);
```

### Objeto de Respuesta

```json
{
    "status":  200,
    "ok":      true,
    "body":    { ... },
    "headers": { "content-type": "application/json" },
    "text":    "...",
    "error":   "..."
}
```

| Campo     | Tipo   | Descripción                                                          |
|-----------|--------|----------------------------------------------------------------------|
| `status`  | `i`    | Código de estado HTTP                                                |
| `ok`      | `b`    | `true` cuando el estado es >= 200 y < 300                            |
| `body`    | `json` | Cuerpo de respuesta parseado (si es JSON); de lo contrario `null`    |
| `headers` | `json` | Cabeceras de respuesta                                               |
| `text`    | `s`    | Cuerpo de respuesta como string sin procesar (siempre disponible)    |
| `error`   | `s`    | Mensaje de error si la petición falló; string vacío si fue correcta  |

`ok` es `true` cuando el estado es >= 200 y < 300. Siempre comprueba `ok` antes de acceder a `body`.

```xcx
json: resp = net.get("https://api.example.com/data");
if (resp.ok) then;
    --- trabajar con resp.body
else;
    s: err;
    if (resp.exists("error")) then;
        resp.bind("error", err);
    end;
    >! "Error: " + s(resp.get("status")) + " | " + err;
end;
```

### Constructor de Bajo Nivel (`net.request`)

```xcx
net.request {
    method  = "POST",
    url     = "https://api.example.com/data",
    headers = ["Authorization" :: "Bearer xyz", "X-App" :: "XCX"],
    body    = my_json,
    timeout = 5000
} as resp;
```

| Campo     | Tipo        | Requerido | Por defecto | Descripción                            |
|-----------|-------------|-----------|-------------|----------------------------------------|
| `method`  | `s`         | Sí        | —           | `"GET"`, `"POST"`, `"PUT"`, etc.       |
| `url`     | `s`         | Sí        | —           | URL completa con esquema               |
| `headers` | `map:s<->s` | No        | `{}`        | Cabeceras de petición adicionales      |
| `body`    | `json`      | No        | `null`      | Ignorado para GET y DELETE             |
| `timeout` | `i`         | No        | `10000`     | Milisegundos                           |

---

## Servidor HTTP

### Directiva `serve`

```xcx
serve: app {
    port    = 8080,
    host    = "0.0.0.0",
    workers = 4,
    routes  = [
        "POST   /api/login"     :: handle_login,
        "GET    /api/user"      :: handle_user,
        "DELETE /api/users"     :: handle_delete,
        "OPTIONS *"             :: handle_options,
        "*"                     :: handle_404
    ]
};
```

| Campo     | Tipo         | Requerido | Por defecto   | Descripción                                        |
|-----------|--------------|-----------|---------------|----------------------------------------------------|
| `port`    | `i`          | Sí        | —             | Número de puerto (1–65535)                         |
| `host`    | `s`          | No        | `"127.0.0.1"` | `"0.0.0.0"` = todas las interfaces                 |
| `workers` | `i`          | No        | `1`           | Fibras por petición                                |
| `routes`  | lista rutas  | Sí        | —             | Se comprueban de arriba a abajo, gana la primera   |

> [!NOTE]
> `serve:` es una **sentencia terminal**. Ningún código posterior a esta directiva se ejecutará.
> El comodín `*` coincide con cualquier método o ruta — colócalo siempre al final.

### Fibras Manejadoras

Cada manejador debe ser una fibra con la firma `fiber name(json: req -> json)`:

```xcx
fiber handle_health(json: req -> json) {
    yield net.respond(200, <<< {"status": "ok"} >>>);
};
```

Un manejador que no llame a `yield net.respond(...)` resulta en un `500 Internal Server Error` automático.

### `net.respond()`

```xcx
yield net.respond(200, my_json);
yield net.respond(201, my_json, ["Location" :: "/users/42"]);
yield net.respond(204, <<< {} >>>);
yield net.respond(404, <<< {"error": "not found"} >>>);
```

| Parámetro | Tipo        | Requerido | Descripción                                                             |
|-----------|-------------|-----------|-------------------------------------------------------------------------|
| status    | `i`         | Sí        | Código de estado HTTP                                                   |
| body      | `json \| s` | Sí        | Objeto JSON o string sin procesar; usa `<<< {} >>>` para JSON vacío    |
| headers   | `map:s<->s` | No        | Cabeceras de respuesta adicionales                                      |

### CORS y Preflight

Por defecto, el motor XCX proporciona soporte CORS automático añadiendo estas cabeceras si faltan en la respuesta:
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

> [!TIP]
> **Práctica Recomendada**: Aunque el motor proporciona valores por defecto, se **recomienda encarecidamente** declarar explícitamente estas cabeceras en tu código. Esto garantiza que tu aplicación sea predecible en diferentes entornos y documenta claramente su política de seguridad.

Para soporte de preflight (`OPTIONS`), usa un manejador dedicado para establecer explícitamente los métodos y cabeceras permitidos:

```xcx
fiber handle_options(json: req -> json) {
    yield net.respond(204, <<< {} >>>, [
        {
            "Access-Control-Allow-Methods": "GET, POST, DELETE, PATCH, OPTIONS",
            "Access-Control-Allow-Headers": "Content-Type, Authorization, X-CSRF-TOKEN"
        }
    ]);
};
```

### Peticiones Concurrentes

Las fibras permiten E/S solapada:

```xcx
fiber fetch(s: url -> json) {
    yield net.get(url);
};

fiber:json: f1 = fetch("https://api.example.com/users");
fiber:json: f2 = fetch("https://api.example.com/posts");
json: r1 = f1.next();
json: r2 = f2.next();
```

### Restricciones de Seguridad

| Restricción                     | Comportamiento                                         |
|---------------------------------|--------------------------------------------------------|
| `localhost` / `127.0.0.1`       | Permitido por defecto                                  |
| `169.254.x.x` (link-local)      | `halt.fatal` — protección SSRF                         |
| `10.x`, `172.16.x`, `192.168.x` | Bloqueado en modo producción                           |
| URLs `file://`                  | `halt.fatal`                                           |
| Tamaño máx. cuerpo respuesta    | 10 MB                                                  |
| Tamaño máx. cuerpo petición     | 10 MB — devuelve 413 sin invocar el manejador          |