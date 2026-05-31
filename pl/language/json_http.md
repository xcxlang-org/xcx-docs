# XCX 3.1 JSON i HTTP

## JSON

Obiekty JSON w XCX 3.1 są **mutowalne**.

### Tworzenie

```xcx
json: config <<< {"port": 8080, "debug": false} >>>;
json: user   <<< {"name": "", "age": 0} >>>;
```

Wartości w literale mogą być placeholderami (`""`, `0`, `false`) do późniejszego wypełnienia przez `.set()`.

### Serializacja z kolekcji
Obiekty i tablice JSON można też tworzyć bezpośrednio z kolekcji XCX za pomocą metody `.toJson()`. Dostępne dla:
- **Map**: Zwraca obiekt JSON.
- **Tabel**: Zwraca tablicę JSON obiektów.

Szczegóły mapowania i zachowania w [Dokumentacji kolekcji](collections.md).

### Parsowanie

```xcx
json: parsed = json.parse(raw_string);
```

> [!CAUTION]
> **Panika przy niepoprawnym JSON (R305)**: Jeśli parsowanie się nie powiedzie, VM kończy działanie natychmiast. Zweryfikuj zawartość ciągu przed parsowaniem.

### Wzorzec mutowalności

Zadeklaruj schemat z wartościami zerowymi, potem wypełnij:

```xcx
json: resp <<< {"token": "", "role": "", "uid": 0, "ok": false} >>>;
resp.set("token", crypto.token(32));
resp.set("role",  "admin");
resp.set("uid",   42);
resp.set("ok",    true);
yield net.respond(200, resp);
```

### Metody JSON

| Metoda                    | Sygnatura             | Zwraca | Opis                                                       |
|---------------------------|-----------------------|--------|------------------------------------------------------------|
| `.exists(path)`           | `(s) → b`             | `b`    | Sprawdza, czy ścieżka istnieje i nie jest null             |
| `.get(path/idx)`          | `(s/i) → json`        | `json` | Pobiera element ze ścieżki lub indeksu                     |
| `.bind(path, var)`        | `(s, ref) → b`        | `b`    | Wyciąga wartość do **wcześniej zadeklarowanej** zmiennej XCX |
| `.set(path, val)`         | `(s, T) → b`          | `b`    | Ustawia wartość na ścieżce; tworzy klucz, jeśli brakuje    |
| `.push(val)`              | `(json) → b`          | `b`    | Dołącza element do węzła tablicy JSON                     |
| `.size()` / `.count()`    | `() → i`              | `i`    | Liczba kluczy (obiekt) lub elementów (tablica)             |
| `.toStr()`                | `() → s`              | `s`    | Serializuje do ciągu JSON                                  |
| `.inject(path, map, tbl)` | `(s, map, table) → b` | `b`    | Masowy import tablicy JSON do tabeli XCX                   |
| `.first()`                | `() → json`           | `json` | Zwraca pierwszy element tablicy JSON; `halt.error`, jeśli pusta |

> [!NOTE]
> **`.push()` na tablicach JSON**: `.push()` działa wyłącznie na węzłach JSON będących tablicami (`[]`). Wywołanie `.push()` na obiekcie JSON (`{}`) powoduje `halt.error`.
>
> ```xcx
> json: data <<< {"items": []} >>>;
> json: obj <<< {"id": 1} >>>;
> data.get("items").push(obj);   --- OK: "items" is an array
> data.push(obj);                --- halt.error: data is an object, not an array
> ```

> [!IMPORTANT]
> **Składnia `.bind()`**: Drugi argument musi być **wcześniej zadeklarowaną zmienną**. Nie można deklarować typu inline.
>
> ```xcx
> --- Wrong
> req.bind("ip", s: ip);
>
> --- Correct
> s: ip;
> req.bind("ip", ip);
> ```

### Notacja ścieżek

Obsługiwane są notacja kropkowa i nawiasowa dla dostępu zagnieżdżonego:

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

### .inject() — import masowy

Importuj tablicę JSON bezpośrednio do tabeli XCX:

```xcx
json: data <<< {"users":[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]} >>>;
table: imported { columns=[uid::i, uname::s] rows=[EMPTY] };
map: mapping { schema=[s<->s] data=["uid"::"id", "uname"::"name"] };
data.inject("users", mapping, imported);
imported.show();
```

### JSON w żądaniach HTTP

W handlerach HTTP `json: req` ma następującą strukturę:

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

## Klient HTTP

### API wysokiego poziomu

| Metoda                 | Sygnatura          | Zwraca |
|------------------------|--------------------|--------|
| `net.get(url)`         | `(s) → json`       | `json` |
| `net.post(url, body)`  | `(s, json) → json` | `json` |
| `net.put(url, body)`   | `(s, json) → json` | `json` |
| `net.delete(url)`      | `(s) → json`       | `json` |

```xcx
json: resp = net.get("https://api.example.com/users");

json: body <<< {"name": "Alice"} >>>;
json: resp = net.post("https://api.example.com/users", body);
```

### Obiekt odpowiedzi

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

| Pole      | Typ    | Opis                                                              |
|-----------|--------|-------------------------------------------------------------------|
| `status`  | `i`    | Kod statusu HTTP                                                  |
| `ok`      | `b`    | `true`, gdy status >= 200 i < 300                                 |
| `body`    | `json` | Sparsowane body odpowiedzi (jeśli JSON); w przeciwnym razie `null` |
| `headers` | `json` | Nagłówki odpowiedzi                                               |
| `text`    | `s`    | Body odpowiedzi jako surowy ciąg (zawsze dostępne)                |
| `error`   | `s`    | Komunikat błędu, jeśli żądanie się nie powiodło; pusty ciąg, jeśli OK |

`ok` jest `true`, gdy status >= 200 i < 300. Zawsze sprawdzaj `ok` przed dostępem do `body`.

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

### Builder niskiego poziomu (`net.request`)

```xcx
net.request {
    method  = "POST",
    url     = "https://api.example.com/data",
    headers = ["Authorization" :: "Bearer xyz", "X-App" :: "XCX"],
    body    = my_json,
    timeout = 5000
} as resp;
```

| Pole      | Typ         | Wymagane | Domyślne | Opis                                    |
|-----------|-------------|----------|----------|-----------------------------------------|
| `method`  | `s`         | Tak      | —        | `"GET"`, `"POST"`, `"PUT"` itd.         |
| `url`     | `s`         | Tak      | —        | Pełny URL ze schematem                  |
| `headers` | `map:s<->s` | Nie      | `{}`     | Dodatkowe nagłówki żądania              |
| `body`    | `json`      | Nie      | `null`   | Ignorowane dla GET i DELETE             |
| `timeout` | `i`         | Nie      | `10000`  | Milisekundy                             |

---

## Serwer HTTP

### Dyrektywa serve

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

| Pole      | Typ          | Wymagane | Domyślne      | Opis                                          |
|-----------|--------------|----------|---------------|-----------------------------------------------|
| `port`    | `i`          | Tak      | —             | Numer portu (1–65535)                         |
| `host`    | `s`          | Nie      | `"127.0.0.1"` | `"0.0.0.0"` = wszystkie interfejsy            |
| `workers` | `i`          | Nie      | `1`           | Włókna na żądanie                             |
| `routes`  | lista tras   | Tak      | —             | Sprawdzane od góry, wygrywa pierwsze dopasowanie |

> [!NOTE]
> `serve:` to **instrukcja terminalna**. Żaden kod po tej dyrektywie nie zostanie wykonany.
> Symbol wieloznaczny `*` pasuje do dowolnej metody lub ścieżki — zawsze umieszczaj go na końcu.

### Handlery (włókna)

Każdy handler musi być włóknem ze sygnaturą `fiber name(json: req -> json)`:

```xcx
fiber handle_health(json: req -> json) {
    yield net.respond(200, <<< {"status": "ok"} >>>);
};
```

Handler, który nie wywołuje `yield net.respond(...)`, skutkuje automatycznym `500 Internal Server Error`.

### net.respond()

```xcx
yield net.respond(200, my_json);
yield net.respond(201, my_json, ["Location" :: "/users/42"]);
yield net.respond(204, <<< {} >>>);
yield net.respond(404, <<< {"error": "not found"} >>>);
```

| Parametr | Typ         | Wymagane | Opis                                                          |
|----------|-------------|----------|---------------------------------------------------------------|
| status   | `i`         | Tak      | Kod statusu HTTP                                              |
| body     | `json \| s` | Tak      | Obiekt JSON lub surowy ciąg; użyj `<<< {} >>>` dla pustego JSON |
| headers  | `map:s<->s` | Nie      | Dodatkowe nagłówki odpowiedzi                                 |

### CORS i preflight

Domyślnie silnik XCX zapewnia automatyczne wsparcie CORS, dodając te nagłówki, jeśli brakuje ich w odpowiedzi:
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

> [!TIP]
> **Zalecana praktyka**: Mimo że silnik dostarcza domyślne wartości, **zdecydowanie zaleca się** jawne deklarowanie tych nagłówków w kodzie. Zapewnia to przewidywalność aplikacji w różnych środowiskach i wyraźnie dokumentuje politykę bezpieczeństwa.

Dla obsługi preflight (`OPTIONS`) użyj dedykowanego handlera, aby jawnie ustawić dozwolone metody i nagłówki:

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

### Równoczesne żądania

Włókna umożliwiają nakładające się operacje I/O:

```xcx
fiber fetch(s: url -> json) {
    yield net.get(url);
};

fiber:json: f1 = fetch("https://api.example.com/users");
fiber:json: f2 = fetch("https://api.example.com/posts");
json: r1 = f1.next();
json: r2 = f2.next();
```

### Ograniczenia bezpieczeństwa

| Ograniczenie                    | Zachowanie                                              |
|---------------------------------|---------------------------------------------------------|
| `localhost` / `127.0.0.1`       | Dozwolone domyślnie                                     |
| `169.254.x.x` (link-local)      | `halt.fatal` — ochrona przed SSRF                       |
| `10.x`, `172.16.x`, `192.168.x` | Zablokowane w trybie produkcyjnym                       |
| URL `file://`                   | `halt.fatal`                                            |
| Maks. rozmiar body odpowiedzi   | 10 MB                                                   |
| Maks. rozmiar body żądania      | 10 MB — zwraca 413 bez wywołania handlera               |
