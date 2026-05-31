# XCX 3.1 — коллекции

## Массивы

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

### Методы массива

| Метод             | Сигнатура    | Возврат | Описание                                                          |
|-------------------|--------------|---------|-------------------------------------------------------------------|
| `.size()`         | `() → i`     | `i`     | Число элементов                                                   |
| `.get(i)`         | `(i) → T`    | `T`     | Элемент с индексом `i` (с нуля); `halt.error` при выходе за границы |
| `.push(val)`      | `(T) → b`    | `b`     | Добавляет элемент в конец                                         |
| `.pop()`          | `() → T`     | `T`     | Удаляет и возвращает последний элемент                            |
| `.insert(i, val)` | `(i, T) → b` | `b`     | Вставка на позицию `i`; `halt.error` при выходе за границы        |
| `.update(i, val)` | `(i, T) → b` | `b`     | Перезапись элемента на позиции `i`                                |
| `.delete(i)`      | `(i) → b`    | `b`     | Удаление элемента на позиции `i`                                  |
| `.find(val)`      | `(T) → i`    | `i`     | Индекс первого вхождения или `-1`                                 |
| `.contains(val)`  | `(T) → b`    | `b`     | Проверка наличия значения                                         |
| `.isEmpty()`      | `() → b`     | `b`     | `true`, если пусто                                                |
| `.clear()`        | `() → b`     | `b`     | Удаляет все элементы                                              |
| `.sort()`         | `() → b`     | `b`     | Сортировка по возрастанию (на месте)                              |
| `.reverse()`      | `() → b`     | `b`     | Обратный порядок (на месте)                                       |
| `.toStr()`        | `() → s`     | `s`     | Сериализация в строку в формате JSON                              |
| `.toJson()`       | `() → json`  | `json`  | Преобразование в нативную JSON-структуру                          |
| `.show()`         | `() → b`     | `b`     | Вывод содержимого в терминал                                      |

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

## Множества

### Домены

| Символ | Тип                 | Пример                        |
|--------|---------------------|-------------------------------|
| `N`    | Натуральные (≥ 0)   | `set:N: s {0, 1, 2}`          |
| `Z`    | Целые               | `set:Z: s {-3, 0, 3}`         |
| `Q`    | Рациональные (float)| `set:Q: s {0.5, 1.0}`         |
| `S`    | Строки              | `set:S: s {"a", "b"}`         |
| `B`    | Логические          | `set:B: s {true, false}`      |
| `C`    | Символы             | `set:C: s {"A",,"Z"}`         |

### Инициализация

Множества можно задать явными значениями или диапазонами. Диапазоны **включительны** с обеих сторон.

```xcx
set:N: small  {1,,5};                  --- {1, 2, 3, 4, 5}
set:N: evens  {0,,100 @step 2};        --- {0, 2, 4, ...}
set:Q: thirds {0.0,,1.0 @step 0.33};
set:C: letters {"A",,"Z"};            --- all uppercase letters
```

Множества **автоматически удаляют дубликаты**.

### Операции над множествами

```xcx
set:N: setA {1,,5};
set:N: setB {3,,7};

set:N: u  = setA UNION setB;
set:N: i  = setA INTERSECTION setB;
set:N: d  = setA DIFFERENCE setB;
set:N: sd = setA SYMMETRIC_DIFFERENCE setB;

--- Unicode symbols are equivalent
setA ∪ setB
setA ∩ setB
setA \ setB
setA ⊕ setB
```

### Методы множества

| Метод          | Сигнатура | Возврат | Описание                                   |
|----------------|-----------|---------|--------------------------------------------|
| `.size()`       | `() → i`  | `i`     | Число элементов                            |
| `.isEmpty()`    | `() → b`  | `b`     | `true`, если пусто                         |
| `.contains(v)`  | `(T) → b` | `b`     | Проверка принадлежности                    |
| `.add(v)`       | `(T) → b` | `b`     | Добавление (дубликаты игнорируются)        |
| `.remove(v)`    | `(T) → b` | `b`     | Удаление (нет эффекта, если нет элемента)  |
| `.clear()`      | `() → b`  | `b`     | Удаляет все элементы                       |
| `.show()`       | `() → b`  | `b`     | Вывод `{elem, elem, ...}` в терминал       |

### Случайный выбор и перебор

```xcx
--- Random selection from a set:
i: picked_set = random.choice from small;

--- Random selection from an array:
array:i: nums {1, 2, 3, 4, 5};
i: picked_arr = random.choice from nums;
```

Выбор из пустого множества или массива возвращает `false`.

### Перебор

```xcx
for p in small do;
    >! p;
end;
```

---

## Карты

```xcx
map: ages {
    schema = [s <-> i]
    data = [ "alice" :: 30, "bob" :: 25 ]
};

--- Empty Map
map: scores {
    schema = [s <-> i]   --- both separators are equivalent (<-> and <=>)
    data = [EMPTY]
};
```

### Методы карты

| Метод            | Сигнатура       | Возврат   | Описание                                    |
|------------------|-----------------|-----------|---------------------------------------------|
| `.size()`        | `() → i`        | `i`       | Число пар ключ–значение                     |
| `.get(key)`      | `(K) → V`       | `V`       | Значение; `halt.error`, если ключа нет      |
| `.contains(key)` | `(K) → b`       | `b`       | Проверка наличия ключа                      |
| `.insert(k, v)`  | `(K, V) → b`    | `b`       | Вставка или перезапись                      |
| `.remove(key)`   | `(K) → b`       | `b`       | Удаление пары; `false`, если ключа нет      |
| `.keys()`        | `() → array:K`  | `array:K` | Массив ключей                               |
| `.values()`      | `() → array:V`  | `array:V` | Массив значений                             |
| `.clear()`       | `() → b`        | `b`       | Удаляет все пары                            |
| `.toStr()`       | `() → s`        | `s`       | Сериализация в JSON-строку                  |
| `.show()`        | `() → b`        | `b`       | Вывод карты в терминал                      |
| `.toJson()`      | `() → json`     | `json`    | Сериализация в JSON-объект                  |

Ключи карты в результирующем JSON преобразуются в строки.

### Сериализация карты (toJson)

#### Сигнатура
`.toJson() → json`

#### Описание
Сериализует карту в JSON-объект. Все ключи преобразуются в строки через `.toString()` в соответствии с требованиями JSON.

```xcx
map: scores {
    schema = [s <-> i]
    data = [ "alice" :: 100, "bob" :: 85 ]
};
json: j = scores.toJson();
```

**Вывод JSON:**
```json
{
    "alice": 100,
    "bob": 85
}
```

#### Особые случаи
- **Пустая карта**: возвращается `{}`.
- **Вложенные структуры**: массивы, другие карты и таблицы рекурсивно сериализуются в JSON.
- **Преобразование ключей**: все ключи становятся строками через стандартный `.toString()`.

#### Соответствие типов
| Тип XCX | Тип JSON  |
|---------|-----------|
| `i`     | `number`  |
| `f`     | `number`  |
| `s`     | `string`  |
| `b`     | `boolean` |
| `date`  | `string` (формат `"YYYY-MM-DD HH:mm:ss"`) |
| `json`  | (без изменений) |

Перед `.get()` используйте `.contains()`:

```xcx
if (ages.contains("alice")) then;
    >! ages.get("alice");
end;
```

---

## Таблицы

Реляционные структуры с опциональными столбцами автоинкремента.

```xcx
table: products {
    columns = [ id :: i @auto, name :: s, price :: f ]
    rows = [ ("Laptop", 2999.99), ("Phone", 1499.50) ]
};

--- Empty Table
table: logs {
    columns = [ id :: i @auto, msg :: s ]
    rows = [EMPTY]
};
```

Модификатор `@auto` на столбце `i` создаёт автоинкрементный ID — он пропускается в `.insert()` и `.add()`.

> [!NOTE]
> Дополнительные атрибуты столбцов (`@pk`, `@unique`, `@optional`, `@default(v)`, `@fk(t.col)`) используются при подключении таблицы к базе данных. Подробности — в [документации по БД](database.md).

### Доступ к строкам

```xcx
products[0].name    --- "Laptop" (sugar for .get(0))
products[1].price   --- 1499.50
```

### Методы таблицы

| Метод                | Сигнатура               | Возврат | Описание                                         |
|----------------------|-------------------------|---------|--------------------------------------------------|
| `.count()`           | `() → i`                | `i`     | Число строк                                      |
| `.get(i)`            | `(i) → row`             | `row`   | Строка с индексом `i`                            |
| `.insert(vals...)`   | `(T...) → b`            | `b`     | Добавляет строку (пропускает `@auto`)           |
| `.add(vals...)`      | `(T...) → b`            | `b`     | Синоним `.insert()` — то же поведение            |
| `.update(i, vals)`   | `(i, [T...]) → b`       | `b`     | Замена значений строки; `@auto` сохраняются      |
| `.delete(i)`         | `(i) → b`               | `b`     | Удаление строки с индексом `i`                   |
| `.where(pred)`       | `(expr) → table`        | `table` | Фильтр — возвращает новую таблицу              |
| `.join(t, pred)`     | `(table, pred) → table` | `table` | Внутреннее соединение с другой таблицей          |
| `.toJson()`          | `() → json`             | `json`  | Все строки как JSON-массив объектов              |
| `.show()`            | `() → b`                | `b`     | Вывод таблицы в ASCII                            |

### Именованные аргументы для `.add()` и `.insert()`

Если у таблицы есть атрибуты столбцов БД, значения можно передавать по имени столбца. Именованные аргументы **опциональны** — позиционные вызовы остаются допустимыми.

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

--- Positional (backward compatible)
users.add("Alice", 25, "", "user");

--- Named
users.add(name = "Alice", age = 25, phone = "", role = "user");

--- Mixed — positional args must come first
users.add("Alice", age = 25, role = "admin");
```

**Разделение пространств имён.** Слева от `=` всегда имя столбца. Справа — выражение из локальной области. Конфликта нет:

```xcx
s: name = "Alice";
users.add(name = name, age = 25);
--- left "name"  = column users.name
--- right "name" = local variable
```

Правила: позиционные аргументы перед именованными; столбцы `@auto` передавать нельзя; пропуск обязательного столбца — ошибка компиляции; дублирование имён столбцов в одном вызове — ошибка. Полная спецификация — в [документации по БД](database.md).

### Фильтрация (where)

```xcx
--- Shorthand syntax (column names usable directly)
table: expensive = products.where(price > 1000.0);
table: named     = products.where(name HAS "Pro");

--- Lambda
table: r = products.where(row -> row.price > 1000.0);

--- Chaining
table: result = products
    .where(price > 1000.0)
    .where(name HAS "Pro");
```

> [!IMPORTANT]
> **Конфликт имён в `.where()` (S301)**: имена столбцов имеют приоритет над локальными переменными в предикате. Если локальная переменная совпадает со столбцом, переименуйте переменную.
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

### Соединения (join)

```xcx
--- Key-based join
table: report = users.join(orders, "id", "user_id");

--- Lambda join
table: custom = tableA.join(tableB, (a, b) -> a.id == b.ref_id);
```

Если в соединённых таблицах есть одинаковые имена столбцов (кроме ключа соединения), в результате столбец получает префикс `{table_name}_`.

### Сериализация (toJson)

#### Сигнатура
`.toJson() → json`

#### Описание
Сериализует все строки таблицы в JSON-массив; каждая строка — объект с ключами по именам столбцов. Столбцы `@auto` включаются в результат.

#### Формат
Всегда возвращается JSON-массив (`[...]`). Пустая таблица — `[]`.

```xcx
table: products {
    columns = [ id :: i @auto, name :: s, price :: f ]
    rows = [ ("Laptop", 2999.99), ("Phone", 1499.50) ]
};

json: result = products.toJson();
```

**Вывод JSON:**
```json
[
    {"id": 1, "name": "Laptop", "price": 2999.99},
    {"id": 2, "name": "Phone",  "price": 1499.50}
]
```

#### Соответствие типов
| Тип XCX         | Тип JSON                                  |
|-----------------|-------------------------------------------|
| `i` / `int`     | `number`                                  |
| `f` / `float`   | `number`                                  |
| `s` / `str`     | `string`                                  |
| `b` / `bool`    | `boolean`                                 |
| `date`          | `string` (формат `"YYYY-MM-DD HH:mm:ss"`) |

#### Особые случаи
- **Пустая таблица**: `[]`.
- **Отфильтрованная таблица**: только строки после `.where()`.
- **Соединённая таблица**: все столбцы результата join.
