# XCX 3.1 — типы данных

## Простые типы

| Символ        | Тип      | По умолчанию | Пример                     |
|---------------|----------|--------------|----------------------------|
| `i` / `int`   | Целое    | `0`          | `42`, `-7`, `0`            |
| `f` / `float` | Вещественное | `0.0`    | `3.14`, `-0.5`, `2.0`      |
| `s` / `str`   | Строка   | `""`         | `"hello"`, `""`            |
| `b` / `bool`  | Логический | `false`    | `true`, `false`            |
| `date`        | Дата     | `1970-01-01` | `date("2024-12-25")`       |
| `json`        | JSON-объект | `null`    | `<<< {"key": "value"} >>>` |

## Составные типы

| Символ       | Тип                         | Пример объявления                            |
|--------------|-----------------------------|----------------------------------------------|
| `array:i`    | Массив целых                | `array:i: nums {1, 2, 3}`                    |
| `array:f`    | Массив вещественных         | `array:f: vals {1.0, 2.5}`                   |
| `array:s`    | Массив строк                | `array:s: words {"a", "b"}`                  |
| `array:b`    | Массив логических           | `array:b: flags {true, false}`               |
| `array:json` | Массив JSON-объектов        | `array:json: items`                          |
| `set:N`      | Множество натуральных        | `set:N: s {1,,10}`                           |
| `set:Z`      | Множество целых             | `set:Z: s {-2, 0, 2}`                        |
| `set:Q`      | Множество рациональных (float) | `set:Q: s {0.5, 1.0, 1.5}`              |
| `set:S`      | Множество строк             | `set:S: s {"a", "b"}`                        |
| `set:B`      | Множество логических        | `set:B: s {true, false}`                     |
| `set:C`      | Множество символов          | `set:C: s {"A",,"Z"}`                        |
| `map`        | Карта ключ–значение         | `map: m { schema = [s <-> i] data = [...] }` |
| `table`      | Реляционная таблица         | `table: t { columns = [...] rows = [...] }`  |
| `database:`  | Подключение к БД            | `database: db { engine = "sqlite", path = "app.db" }` |
| `fiber:T`    | Типизированный файбер       | `fiber:b: f = my_fiber(arg)`                 |
| `fiber:`     | Пустой файбер               | `fiber: f = my_void_fiber(arg)`              |

### array:json

`array:json` — массив элементов типа `json`. В основном используется при работе с HTTP-ответами, возвращающими массивы JSON-объектов.

```xcx
--- Declare an empty JSON array
array:json: items;

--- Iterate over a JSON array (e.g. from a network response)
i: size = resp.body.size();
i: idx = 0;
while (idx < size) do;
    json: item = resp.body.get(idx);
    s: name; item.bind("name", name);
    >! name;
    idx = idx + 1;
end;

--- Add an element
json: obj <<< {"key": "value"} >>>;
items.push(obj);
```

Методы `array:json` совпадают с другими типами массивов (`.push()`, `.pop()`, `.get(i)`, `.size()` и т.д.).

### database:

`database:` представляет подключение к реляционной базе данных. Объявляется блоком конфигурации и используется через встроенные методы. Полный API — в [документации по базам данных](database.md).

```xcx
database: app {
    engine = "sqlite",
    path   = "app.db"
};

yield app.sync(users);
table: all = yield app.fetch(users);
```

## Значения по умолчанию

```xcx
int:   def_int;     --- 0 (alias for i)
float: def_float;   --- 0.0 (alias for f)
str:   def_str;     --- "" (alias for s)
bool:  def_bool;    --- false (alias for b)
```

## Приведение типов

```xcx
f: x = 3.7;
i: n = i(x);       --- 3 (truncate toward zero)

i: m = 42;
f: y = f(m);       --- 42.0

i: num = 99;
s: str = s(num);   --- "99"
```

> [!NOTE]
> Преобразование `b → i` **намеренно запрещено**. `true + 1` — ошибка типа, чтобы избежать логических ошибок, типичных для языков семейства C.
