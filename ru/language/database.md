# XCX 3.1 — база данных

XCX 3.1 добавляет встроенную поддержку реляционных БД через тип `database:` и набор методов. В версии 3.1 поддерживается только **SQLite**.

---

## Содержание

1. [Объявление подключения (`database:`)](#1-объявление-подключения-database)
2. [Атрибуты столбцов таблицы](#2-атрибуты-столбцов-таблицы)
3. [Именованные аргументы](#3-именованные-аргументы)
4. [Захват результата (`as`)](#4-захват-результата-as)
5. [Ограничения области видимости в файберах (Windows)](#5-ограничения-области-видимости-в-файберах-windows)
6. [Справочник API](#6-справочник-api)
   - 6.1 [DDL](#61-ddl)
   - 6.2 [Операции записи](#62-операции-записи)
   - 6.3 [Операции чтения](#63-операции-чтения)
   - 6.4 [Операции удаления](#64-операции-удаления)
   - 6.5 [Транзакции](#65-транзакции)
   - 6.6 [Прочее](#66-прочее)
7. [Соответствие типов (XCX ↔ SQL)](#7-соответствие-типов-xcx--sql)
8. [Ограничения безопасности](#8-ограничения-безопасности)
9. [Полный пример](#9-полный-пример)

---

## 1. Объявление подключения (`database:`)

```xcx
database: app {
    engine  = "sqlite",
    path    = "app.db"
};
```

### Поля

| Поле       | Тип | Обязательно | По умолчанию | Описание                                           |
|------------|-----|-------------|--------------|----------------------------------------------------|
| `engine`   | `s` | да          | —            | Движок БД. Сейчас только `"sqlite"`.               |
| `path`     | `s` | да          | —            | Путь к файлу `.db` (относительно корня проекта).   |
| `timeout`  | `i` | нет         | `5000`       | Таймаут операции в мс.                             |
| `readonly` | `b` | нет         | `false`      | Режим только чтения.                               |

### Поведение

- Подключения **ленивые** — открываются при первом использовании.
- При ошибке подключения (например, нет прав) автоматически вызывается `halt.error`.
- SQLite создаёт файл БД, если его нет (кроме `readonly = true`).
- Допускается несколько одновременных подключений — у каждого своё имя.

```xcx
database: main { engine = "sqlite", path = "main.db" };
database: logs { engine = "sqlite", path = "logs.db" };

yield main.sync(users);
yield logs.sync(events);
```

### Модель ввода-вывода

Все операции БД, затрагивающие диск или драйвер, — операции I/O:

| Контекст        | Поведение                |
|-----------------|--------------------------|
| Внутри файбера  | требуется `yield`        |
| Вне файбера     | синхронная блокировка    |

Относится к **чтению** (`fetch`, `query`, `queryRaw`), **записи** (`push`, `save`, `insert`, `exec`), **удалению** (`remove`, `exec`) и **DDL** (`sync`, `drop`).

**Исключение — методы без `yield`.** Методы, работающие только с состоянием транзакции или метаданными:

- `db.has()`, `db.close()`, `db.isOpen()`
- `db.begin()`, `db.commit()`, `db.rollback()`

> `begin()`, `commit()` и `rollback()` не трогают данные — только состояние транзакции в драйвере.

---

## 2. Атрибуты столбцов таблицы

Эти атрибуты расширяют объявление блока `table:` и используются модулем базы данных.

### Атрибуты

| Атрибут         | Описание                                                         | Эквивалент SQL      |
|-----------------|------------------------------------------------------------------|---------------------|
| `@pk`           | Первичный ключ. Можно сочетать с `@auto`.                       | `PRIMARY KEY`       |
| `@unique`       | Значение уникально в столбце.                                    | `UNIQUE`            |
| `@optional`     | В SQL допускается `NULL`. При чтении в XCX — значение по умолчанию типа. | `NULL`      |
| `@default(v)`   | Значение по умолчанию в SQL.                                     | `DEFAULT v`         |
| `@fk(t.col)`    | Внешний ключ на столбец `col` таблицы `t`.                      | `REFERENCES t(col)` |

### Пример

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

### Поведение `@optional`

В XCX нет отдельного типа `null`. Столбцы `@optional` в SQL могут хранить `NULL`, но при чтении в XCX значение заменяется значением по умолчанию типа:

| Тип XCX | При SQL NULL |
|---------|--------------|
| `i`     | `0`          |
| `f`     | `0.0`        |
| `s`     | `""`         |
| `b`     | `false`      |

> **Примечание:** `@optional` только разрешает `NULL` на стороне SQL. XCX не различает `NULL` и значение по умолчанию — эта информация теряется при чтении. Если логике нужно различать случаи, храните флаг в отдельном столбце.

### Правила пропуска столбцов для `add()` и `insert()`

| Атрибут столбца          | Можно пропустить? | Что попадает в SQL           |
|--------------------------|-------------------|------------------------------|
| `@auto`                  | да (всегда)       | авто-сгенерированное значение|
| `@default(v)`            | да                | значение `v` из `DEFAULT`    |
| `@optional`              | да                | `NULL`                       |
| `@optional @default(v)`  | да                | значение `v` из `DEFAULT`    |
| без атрибутов            | **нет**           | ошибка компиляции            |

> Столбцы `@auto` нельзя передавать явно — ни позиционно, ни по имени. Попытка — **ошибка компиляции**.

---

## 3. Именованные аргументы

Именованные аргументы позволяют передавать значения в `table.add()`, `table.insert()` и `db.insert()` **по имени столбца**, а не по позиции. Это опциональное дополнение — позиционные вызовы остаются допустимыми.

Работают **только** для вставки в таблицы, не для пользовательских функций, файберов и других встроенных методов. Имена столбцов известны компилятору из схемы `table:`.

### Синтаксис

```xcx
--- Positional (backward compatible)
users.add("Alice", 25, "", "user");

--- Named
users.add(name = "Alice", age = 25, phone = "", role = "user");

--- With db.insert()
yield app.insert(users, name = "Alice", age = 25) as saved;
```

**Разделение пространств имён.** Слева от `=` — имя столбца. Справа — выражение из локальной области:

```xcx
s: name = "Alice";
users.add(name = name, age = 25);
--- left "name"  = column users.name
--- right "name" = local variable
```

**Порядок вычисления.** Правые части вычисляются слева направо в порядке записи:

```xcx
users.add(name = get_name(), age = get_age());
--- get_name() is called before get_age()
```

### Смешение позиционных и именованных

Позиционные аргументы должны идти **перед** именованными.

```xcx
--- OK — positional before named
users.add("Alice", age = 25, role = "admin");

--- COMPILE ERROR — named before positional
users.add(name = "Alice", 25, "admin");
```

**Правило присвоения.** Позиционные аргументы назначаются столбцам слева направо в порядке объявления, **пропуская `@auto`**, пока не закончатся. Именованные заполняют оставшиеся столбцы по имени.

```xcx
--- "Alice" → name, rest named
users.add("Alice", age = 25, role = "admin");
--- phone omitted → NULL (@optional)

--- "Alice", 25 → name, age; role named, phone omitted
users.add("Alice", 25, role = "mod");
```

Позиционные аргументы назначаются **без пропусков**. Пропустить средний столбец можно только именованным аргументом или полным пропуском (если столбец опционален):

```xcx
--- COMPILE ERROR — compiler assigns: name="Alice", age="" — type mismatch
--- there is no way to "skip" age positionally
users.add("Alice", "", "user");
```

Один столбец и позиционно, и по имени — **ошибка компиляции**:

```xcx
--- COMPILE ERROR — name provided twice
users.add("Alice", name = "Bob", age = 25);
```

### Правила компиляции

| Правило                         | Поведение           |
|---------------------------------|---------------------|
| Передан столбец `@auto`         | ошибка компиляции   |
| Неизвестное имя столбца         | ошибка компиляции   |
| Дублирование имени столбца      | ошибка компиляции   |
| Пропущен обязательный столбец   | ошибка компиляции   |
| Именованный аргумент перед позиционным | ошибка компиляции |

**Таблица полноты:**

| Атрибут столбца         | Можно пропустить? | Значение при пропуске              |
|-------------------------|-------------------|------------------------------------|
| `@auto`                 | да (всегда)       | авто-сгенерированное значение      |
| `@default(v)`           | да                | значение `v` из `DEFAULT`          |
| `@optional`             | да                | `NULL` (→ значение по умолчанию XCX при чтении) |
| `@optional @default(v)` | да                | значение `v` из `DEFAULT`          |
| без атрибутов           | **нет**           | ошибка компиляции                  |

```xcx
--- OK — role has @default("user"), phone is @optional
users.add(name = "Alice", age = 25);

--- COMPILE ERROR — age is required
users.add(name = "Alice");
```

---

## 4. Захват результата (`as`)

`as` — общий механизм XCX для захвата результата блочной операции:

```xcx
--- HTTP request
net.request { method = "GET", url = "..." } as resp;

--- DB operations
yield app.insert(users, name = "Alice", age = 25) as saved;
yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
```

### Объект результата записи/удаления

Операции `db.insert()`, `db.save()`, `db.push()` и `db.exec()` возвращают объект с двумя полями:

| Поле       | Тип | Описание                                              |
|------------|-----|-------------------------------------------------------|
| `affected` | `i` | Число изменённых / вставленных строк                  |
| `insertId` | `i` | ID последней вставленной записи (`@auto @pk`); `0`, если вставки не было |

```xcx
yield app.insert(users, name = "Alice", age = 25) as saved;
>! "New user ID: " + s(saved.insertId);

yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
>! "Deleted: " + s(res.affected);
```

`as` опционален — если результат не нужен, его можно опустить:

```xcx
yield app.save(users);
```

### Семантика `affected` и `insertId`

| Операция                   | `affected`              | `insertId`                |
|----------------------------|-------------------------|---------------------------|
| `insert()` — одна строка   | `1`                     | ID вставленной строки     |
| `push()` — несколько строк | число вставленных строк | ID последней вставки      |
| `save()` — upsert (insert) | `1`                     | ID вставленной строки     |
| `save()` — upsert (update) | `1`                     | `0`                       |
| `exec(DELETE ...)`         | число удалённых строк   | `0`                       |
| `exec(UPDATE ...)`         | число обновлённых строк | `0`                       |

> При сбое операции (нарушение ограничения, нет подключения, `readonly`) вызывается `halt.error`, и код с `as` не выполняется. Если код доходит до результата, объект всегда корректен.

---

## 5. Ограничения области видимости в файберах (Windows)

В XCX 3.1 на Windows инициализация локальной переменной файбера напрямую результатом запроса к БД иногда приводит к ошибке `Undefined variable` [S101] на следующих строках.

### Рекомендуемый обходной путь

Чтобы переменная была видна во всех ветках внутри файбера, **сначала объявите тип**, затем присвойте результат БД отдельной инструкцией.

```xcx
fiber run(-> b) {
    --- Recommended pattern
    b: exists_before;
    exists_before = db.has(logs);
    
    if (exists_before) then;
        >! "Table exists";
    end;
}
```

Так переменная регистрируется в таблице символов до обращения анализатора. Ограничение планируется устранить архитектурно в XCX 4.0.

---

## 6. Справочник API

### 6.1 DDL

```xcx
--- Creates the SQL table if it does not exist (based on XCX table schema)
yield app.sync(users);

--- Drops the SQL table
yield app.drop(users);

--- Checks if table exists in the database (no data touched — no yield)
b: exists = app.has(users);
```

### 6.2 Операции записи

```xcx
--- INSERT all rows from a local XCX table into SQL
yield app.push(users);

--- INSERT OR UPDATE (upsert) — requires @pk; no @pk = compile error
--- Implementation: INSERT INTO ... ON CONFLICT(pk) DO UPDATE SET ...
yield app.save(users);

--- INSERT a single row — positional or named
yield app.insert(users, "Alice", 25) as saved;
yield app.insert(users, name = "Alice", age = 25) as saved;
```

При ошибке (нарушение `@unique`, `@fk`, нет подключения) операция вызывает `halt.error`.

### 6.3 Операции чтения

```xcx
--- SELECT * — returns a native XCX table
table: all_users = yield app.fetch(users);

--- SELECT with XCX-style filter (compiles to SQL WHERE)
table: adults = yield app.fetch(users).where(age > 18);

--- SELECT with raw SQL — requires schema hint as first argument
table: result = yield app.query(users, "SELECT * FROM users WHERE age > ?", [18]);

--- Raw SQL without schema — always returns a JSON array
json: raw = yield app.queryRaw("SELECT COUNT(*) as n FROM users");
--- raw = [{"n": 42}]

--- Get the first row
json: row = yield app.queryRaw("SELECT COUNT(*) as n FROM users").first();
--- row = {"n": 42}
--- halt.error if result is empty
```

#### `.first()` для результата `queryRaw`

`.first()` возвращает первый элемент JSON-массива как один объект `json`. При пустом результате — `halt.error`. Типично для агрегатов (`COUNT`, `SUM`, `MAX`) и выборок не более одной строки.

```xcx
json: row = yield app.queryRaw("SELECT COUNT(*) as n FROM users").first();
i: total; row.bind("n", total);
>! "Users: " + s(total);
```

Если результат может быть пустым, проверьте размер перед `.first()`:

```xcx
json: results = yield app.queryRaw("SELECT * FROM users WHERE age > ?", [100]);
if (results.size() > 0) then;
    json: row = results.get(0);
end;
```

#### Поддерживаемые операторы в `.where()`

Фильтр `.where()` компилируется в SQL `WHERE`. Поддерживается ограниченное подмножество выражений XCX:

| Категория     | Поддерживается                   |
|---------------|----------------------------------|
| Сравнения     | `==`, `!=`, `<`, `<=`, `>`, `>=` |
| Логика        | `AND`, `OR`, `NOT`               |
| Значения      | литералы, локальные переменные   |
| Левые операнды| только имена столбцов            |

Вызовы функций, методы строк и вложенные выражения в `.where()` с `db.fetch()` не поддерживаются — **ошибка компиляции**.

```xcx
--- OK
table: r = yield app.fetch(users).where(age > 18 AND role == "admin");

--- COMPILE ERROR — string method not allowed in where() with fetch
table: r = yield app.fetch(users).where(name.lower() == "alice");
```

Цепочка `.where()` объединяет условия через `AND`:

```xcx
table: r = yield app.fetch(users)
    .where(age > 18)
    .where(role == "admin");
--- SQL equivalent: WHERE age > 18 AND role = 'admin'
```

#### `db.fetch()` и локальные `rows`

`db.fetch()` читает только данные из SQL — локальные `rows` в блоке `table:` игнорируются. Локальное определение таблицы служит только подсказкой схемы.

```xcx
table: users {
    columns = [ id :: i @auto @pk, name :: s ]
    rows = [ ("Alice") ]   --- these rows do NOT go into fetch
};

yield app.sync(users);

--- Returns only rows stored in app.db
table: all = yield app.fetch(users);
```

Чтобы загрузить локальные `rows` в БД, сначала вызовите `db.push(users)`, затем `db.fetch()`.

### 6.4 Операции удаления

```xcx
--- DELETE with filter — .where() is required (D401)
yield app.remove(users).where(age < 18);

--- Delete all rows
yield app.truncate(users);

--- Raw DELETE with parameters
yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
```

> **`db.remove()` без `.where()` — ошибка компиляции (D401).** Чтобы удалить все строки, явно используйте `db.truncate(users)`.

### 6.5 Транзакции

```xcx
app.begin();
yield app.save(users);
yield app.save(posts);
app.commit();
```

> Если любая операция в транзакции вызывает `halt.error`, транзакция **автоматически откатывается** до прерывания кадра. Подключение возвращается в состояние до `begin()`.

`rollback()` — для явной отмены транзакции в условной логике до ошибки:

```xcx
app.begin();
b: valid = validate_users(users);
if (NOT valid) then;
    app.rollback();
end;
yield app.save(users);
app.commit();
```

### 6.6 Прочее

```xcx
app.close();
b: alive = app.isOpen();
```

---

## 7. Соответствие типов (XCX ↔ SQL)

| Тип XCX | Тип SQL (SQLite)               |
|---------|--------------------------------|
| `i`     | `INTEGER`                      |
| `f`     | `REAL`                         |
| `s`     | `TEXT`                         |
| `b`     | `INTEGER` (0 / 1)              |
| `date`  | `TEXT` (`YYYY-MM-DD HH:mm:ss`) |

---

## 8. Ограничения безопасности

| Ограничение                    | Поведение                                              |
|--------------------------------|--------------------------------------------------------|
| Путь с `..`                    | `halt.fatal` — обход каталогов                         |
| Абсолютный путь                | `halt.fatal`                                           |
| Сырой SQL с непроверенным вводом | ответственность разработчика — используйте `?`     |
| `readonly = true` + DML        | `halt.error`                                           |
| `save()` без `@pk`             | ошибка компиляции                                      |
| `remove()` без `.where()`      | ошибка компиляции (D401)                               |

Всегда используйте подготовленные запросы с `?`:

```xcx
--- WRONG
app.exec("DELETE FROM users WHERE name = '" + name + "'");

--- CORRECT
app.exec("DELETE FROM users WHERE name = ?", [name]);
```

---

## 9. Полный пример

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

    --- Named arguments — readable and order-independent
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
