# XCX 3.1 Tipos de Datos

## Tipos Simples

| Símbolo       | Tipo         | Por defecto  | Ejemplo                    |
|---------------|-------------|--------------|----------------------------|
| `i` / `int`   | Entero      | `0`          | `42`, `-7`, `0`            |
| `f` / `float` | Flotante    | `0.0`        | `3.14`, `-0.5`, `2.0`      |
| `s` / `str`   | Cadena      | `""`         | `"hello"`, `""`            |
| `b` / `bool`  | Booleano    | `false`      | `true`, `false`            |
| `date`        | Fecha       | `1970-01-01` | `date("2024-12-25")`       |
| `json`        | Objeto JSON | `null`       | `<<< {"key": "value"} >>>` |

## Tipos Complejos

| Símbolo      | Tipo                          | Ejemplo de declaración                       |
|--------------|------------------------------|----------------------------------------------|
| `array:i`    | Array de enteros             | `array:i: nums {1, 2, 3}`                    |
| `array:f`    | Array de flotantes           | `array:f: vals {1.0, 2.5}`                   |
| `array:s`    | Array de cadenas             | `array:s: words {"a", "b"}`                  |
| `array:b`    | Array de booleanos           | `array:b: flags {true, false}`               |
| `array:json` | Array de objetos JSON        | `array:json: items`                          |
| `set:N`      | Conjunto de naturales        | `set:N: s {1,,10}`                           |
| `set:Z`      | Conjunto de enteros          | `set:Z: s {-2, 0, 2}`                        |
| `set:Q`      | Conjunto de racionales (float)| `set:Q: s {0.5, 1.0, 1.5}`                  |
| `set:S`      | Conjunto de cadenas          | `set:S: s {"a", "b"}`                        |
| `set:B`      | Conjunto de booleanos        | `set:B: s {true, false}`                     |
| `set:C`      | Conjunto de caracteres       | `set:C: s {"A",,"Z"}`                        |
| `map`        | Mapa clave-valor             | `map: m { schema = [s <-> i] data = [...] }` |
| `table`      | Tabla relacional             | `table: t { columns = [...] rows = [...] }`  |
| `database:`  | Conexión a base de datos     | `database: db { engine = "sqlite", path = "app.db" }` |
| `fiber:T`    | Fiber con tipo               | `fiber:b: f = my_fiber(arg)`                 |
| `fiber:`     | Fiber sin tipo (void)        | `fiber: f = my_void_fiber(arg)`              |

### array:json

`array:json` es un array que almacena elementos de tipo `json`. Se usa principalmente al trabajar con respuestas HTTP que devuelven arrays de objetos JSON.

```xcx
--- Declarar un array JSON vacío
array:json: items;

--- Iterar sobre un array JSON (por ejemplo, de una respuesta de red)
i: size = resp.body.size();
i: idx = 0;
while (idx < size) do;
    json: item = resp.body.get(idx);
    s: name; item.bind("name", name);
    >! name;
    idx = idx + 1;
end;

--- Agregar un elemento
json: obj <<< {"key": "value"} >>>;
items.push(obj);
```

Los métodos de `array:json` son idénticos a los de otros tipos de array (`.push()`, `.pop()`, `.get(i)`, `.size()`, etc.).

### database:

`database:` representa una conexión a una base de datos relacional. Se declara con un bloque de configuración y se utiliza mediante sus métodos integrados. Consulta la [Documentación de Base de Datos](database.md) para la API completa.

```xcx
database: app {
    engine = "sqlite",
    path   = "app.db"
};

yield app.sync(users);
table: all = yield app.fetch(users);
```

## Valores por Defecto

```xcx
int:   def_int;     --- 0 (alias para i)
float: def_float;   --- 0.0 (alias para f)
str:   def_str;     --- "" (alias para s)
bool:  def_bool;    --- false (alias para b)
```

## Conversión de Tipos

```xcx
f: x = 3.7;
i: n = i(x);       --- 3 (trunca hacia cero)

i: m = 42;
f: y = f(m);       --- 42.0

i: num = 99;
s: str = s(num);   --- "99"
```

> [!NOTE]
> La conversión `b → i` está **bloqueada intencionalmente**. `true + 1` es un error de tipos para prevenir errores lógicos comunes en lenguajes de la familia C.