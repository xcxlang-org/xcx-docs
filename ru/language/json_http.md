# XCX 3.1 — JSON и HTTP

## JSON

JSON-объекты в XCX 3.1 **изменяемы**.

### Создание

```xcx
json: config <<< {"port": 8080, "debug": false} >>>;
json: user   <<< {"name": "", "age": 0} >>>;
```

В литерале можно использовать заглушки (`""`, `0`, `false`) и заполнить их позже через `.set()`.

### Сериализация из коллекций
JSON-объекты и массивы можно создавать из коллекций XCX методом `.toJson()`:
- **Карты**: JSON-объект.
- **Таблицы**: JSON-массив объектов.

Подробнее о сопоставлении типов — в [документации по коллекциям](collections.md).

### Разбор

```xcx
json: parsed = json.parse(raw_string);
```

> [!CAUTION]
> **Паника при неверном JSON (R305)**: при ошибке разбора VM завершается немедленно. Проверяйте содержимое строки перед разбором.

### Шаблон изменяемости

Объявите схему с нулевыми значениями, затем заполните:

```xcx
json: resp <<< {"token": "", "role": "", "uid": 0, "ok": false} >>>;
resp.set("token", crypto.token(32));
resp.set("role",  "admin");
resp.set("uid",   42);
resp.set("ok",    true);
yield net.respond(200, resp);
```

### Методы JSON

| Метод                     | Сигнатура             | Возврат | Описание                                              |
|---------------------------|-----------------------|---------|-------------------------------------------------------|
| `.exists(path)`           | `(s) → b`             | `b`     | Путь существует и не null                             |
| `.get(path/idx)`          | `(s/i) → json`        | `json`  | Элемент по пути или индексу                           |
| `.bind(path, var)`        | `(s, ref) → b`        | `b`     | Извлечение в **заранее объявленную** переменную XCX   |
| `.set(path, val)`         | `(s, T) → b`          | `b`     | Установка значения; создаёт ключ при отсутствии       |
| `.push(val)`              | `(json) → b`          | `b`     | Добавление в узел JSON-массива                        |
| `.size()` / `.count()`    | `() → i`              | `i`     | Число ключей (объект) или элементов (массив)          |
| `.toStr()`                | `() → s`              | `s`     | Сериализация в JSON-строку                            |
| `.inject(path, map, tbl)` | `(s, map, table) → b` | `b`     | Массовый импорт JSON-массива в таблицу XCX            |
| `.first()`                | `() → json`           | `json`  | Первый элемент JSON-массива; `halt.error`, если пусто |

> [!NOTE]
> **`.push()` для JSON-массивов**: `.push()` работает только с узлами-массивами (`[]`). Вызов на JSON-объекте (`{}`) даёт `halt.error`.
>
> ```xcx
> json: data <<< {"items": []} >>>;
> json: obj <<< {"id": 1} >>>;
> data.get("items").push(obj);   --- OK: "items" is an array
> data.push(obj);                --- halt.error: data is an object, not an array
> ```

> [!IMPORTANT]
> **Синтаксис `.bind()`**: второй аргумент — **уже объявленная переменная**. Нельзя объявить тип в строке вызова.
>
> ```xcx
> --- Wrong
> req.bind("ip", s: ip);
>
> --- Correct
> s: ip;
> req.bind("ip", ip);
> ```

### Нотация пути

Поддерживаются точечная и скобочная нотация для вложенного доступа:

```xcx
--- Nested field
json: cfg <<< {"server": {"host": "localhost"}} >>>;
s: host;
cfg.bind("server.host", host);

--- Array index
json: resp <<< {"items": []} >>>;
resp.set("items[0]", first_item);
resp.set("items[1]", second_item);
```

### .inject() — массовый импорт

Импорт JSON-массива напрямую в таблицу XCX:

```xcx
json: data <<< {"users":[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]} >>>;
table: imported { columns=[uid::i, uname::s] rows=[EMPTY] };
map: mapping { schema=[s<->s] data=["uid"::"id", "uname"::"name"] };
data.inject("users", mapping, imported);
imported.show();
```

### JSON в HTTP-запросах

В обработчиках HTTP `json: req` имеет такую структуру:

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

## HTTP-клиент

### Высокоуровневый API

| Метод                 | Сигнатура          | Возврат |
|-----------------------|--------------------|---------|
| `net.get(url)`        | `(s) → json`       | `json`  |
| `net.post(url, body)` | `(s, json) → json` | `json`  |
| `net.put(url, body)`  | `(s, json) → json` | `json`  |
| `net.delete(url)`     | `(s) → json`       | `json`  |

```xcx
json: resp = net.get("https://api.example.com/users");

json: body <<< {"name": "Alice"} >>>;
json: resp = net.post("https://api.example.com/users", body);
```

### Объект ответа

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

| Поле      | Тип    | Описание                                              |
|-----------|--------|-------------------------------------------------------|
| `status`  | `i`    | HTTP-код состояния                                    |
| `ok`      | `b`    | `true`, если status >= 200 и < 300                    |
| `body`    | `json` | Разобранное тело (если JSON); иначе `null`            |
| `headers` | `json` | Заголовки ответа                                      |
| `text`    | `s`    | Тело как сырая строка (всегда доступно)               |
| `error`   | `s`    | Сообщение об ошибке; пустая строка при успехе         |

`ok` равен `true`, если status >= 200 и < 300. Перед доступом к `body` всегда проверяйте `ok`.

```xcx
json: resp = net.get("https://api.example.com/data");
if (resp.ok) then;
    --- working with resp.body
else;
    s: err;
    if (resp.exists("error")) then;
        resp.bind("error", err);
    end;
    >! "Error: " + s(resp.get("status")) + " | " + err;
end;
```

### Низкоуровневый построитель (`net.request`)

```xcx
net.request {
    method  = "POST",
    url     = "https://api.example.com/data",
    headers = ["Authorization" :: "Bearer xyz", "X-App" :: "XCX"],
    body    = my_json,
    timeout = 5000
} as resp;
```

| Поле      | Тип         | Обязательно | По умолчанию | Описание                              |
|-----------|-------------|-------------|--------------|---------------------------------------|
| `method`  | `s`         | Да          | —            | `"GET"`, `"POST"`, `"PUT"` и т.д.     |
| `url`     | `s`         | Да          | —            | Полный URL со схемой                  |
| `headers` | `map:s<->s` | Нет         | `{}`         | Дополнительные заголовки запроса      |
| `body`    | `json`      | Нет         | `null`       | Игнорируется для GET и DELETE         |
| `timeout` | `i`         | Нет         | `10000`      | Миллисекунды                          |

---

## HTTP-сервер

### Директива serve

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

| Поле      | Тип          | Обязательно | По умолчанию  | Описание                                      |
|-----------|--------------|-------------|---------------|-----------------------------------------------|
| `port`    | `i`          | Да          | —             | Порт (1–65535)                                |
| `host`    | `s`          | Нет         | `"127.0.0.1"` | `"0.0.0.0"` — все интерфейсы                  |
| `workers` | `i`          | Нет         | `1`           | Файберов на запрос                            |
| `routes`  | список маршрутов | Да     | —             | Сверху вниз, побеждает первое совпадение      |

> [!NOTE]
> `serve:` — **терминальная** инструкция. Код после неё не выполняется.
> Подстановка `*` для любого метода или пути — всегда в конце списка.

### Обработчики (файберы)

Каждый обработчик — файбер с сигнатурой `fiber name(json: req -> json)`:

```xcx
fiber handle_health(json: req -> json) {
    yield net.respond(200, <<< {"status": "ok"} >>>);
};
```

Обработчик без `yield net.respond(...)` даёт автоматический `500 Internal Server Error`.

### net.respond()

```xcx
yield net.respond(200, my_json);
yield net.respond(201, my_json, ["Location" :: "/users/42"]);
yield net.respond(204, <<< {} >>>);
yield net.respond(404, <<< {"error": "not found"} >>>);
```

| Параметр | Тип         | Обязательно | Описание                                              |
|----------|-------------|-------------|-------------------------------------------------------|
| status   | `i`         | Да          | HTTP-код                                              |
| body     | `json \| s` | Да          | JSON или строка; пустой JSON — `<<< {} >>>`           |
| headers  | `map:s<->s` | Нет         | Дополнительные заголовки ответа                       |

### CORS и preflight

По умолчанию движок XCX добавляет заголовки CORS, если их нет в ответе:
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

> [!TIP]
> **Рекомендация**: несмотря на значения по умолчанию, **желательно** явно задавать эти заголовки в коде — так поведение предсказуемо в разных средах и политика безопасности документирована.

Для preflight (`OPTIONS`) используйте отдельный обработчик:

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

### Параллельные запросы

Файберы позволяют перекрывать I/O:

```xcx
fiber fetch(s: url -> json) {
    yield net.get(url);
};

fiber:json: f1 = fetch("https://api.example.com/users");
fiber:json: f2 = fetch("https://api.example.com/posts");
json: r1 = f1.next();
json: r2 = f2.next();
```

### Ограничения безопасности

| Ограничение                     | Поведение                                            |
|---------------------------------|------------------------------------------------------|
| `localhost` / `127.0.0.1`       | Разрешено по умолчанию                               |
| `169.254.x.x` (link-local)      | `halt.fatal` — защита от SSRF                        |
| `10.x`, `172.16.x`, `192.168.x` | Заблокировано в production                           |
| URL `file://`                   | `halt.fatal`                                         |
| Макс. размер тела ответа        | 10 МБ                                                |
| Макс. размер входящего запроса  | 10 МБ — 413 без вызова обработчика                   |
