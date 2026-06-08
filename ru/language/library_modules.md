# XCX 3.1 — стандартная библиотека и модули

## Встроенные модули

### crypto

Криптографические утилиты.

| Метод                               | Возврат | Описание                                         |
|-------------------------------------|---------|--------------------------------------------------|
| `crypto.hash(data, "bcrypt")`       | `s`     | Хеш пароля через bcrypt                          |
| `crypto.hash(data, "argon2")`       | `s`     | Хеш пароля через argon2 (рекомендуется)          |
| `crypto.hash(data, "base64_encode")`| `s`     | Кодирование данных/строки в Base64               |
| `crypto.hash(data, "base64_decode")`| `s`    | Декодирование Base64 в двоичные данные/строку    |
| `crypto.verify(password, hash, algo)` | `b`   | `true`, если пароль совпадает с хешем            |
| `crypto.token(length)`              | `s`     | Случайный hex-токен заданной длины               |

```xcx
s: hash  = crypto.hash(password, "bcrypt");
s: hash2 = crypto.hash(password, "argon2");
b: valid = crypto.verify(password, hash2, "argon2");
s: token = crypto.token(32);
```

### store (файловый ввод-вывод)

Все пути должны быть **относительными** к корню проекта. Абсолютные пути или обход каталогов (`..`) вызывают `halt.fatal`.

| Метод               | Сигнатура                 | Возврат   | Описание                                           |
|---------------------|---------------------------|-----------|----------------------------------------------------|
| `store.write(p, c)` | `(s, s) → b`              | `b`       | Перезапись файла; при необходимости создаёт каталоги |
| `store.read(p)`     | `(s) → s`                 | `s`       | Содержимое файла; `halt.fatal`, если файла нет     |
| `store.append(p, c)`| `(s, s) → b`              | `b`       | Дозапись; создаёт файл при отсутствии              |
| `store.exists(p)`   | `(s) → b`                 | `b`       | Проверка существования без побочных эффектов       |
| `store.delete(p)`   | `(s) → b`                 | `b`       | Удаление файла или каталога (рекурсивно)           |
| `store.list(p)`     | `(s) → array:s`           | `array:s` | Список файлов и папок                              |
| `store.isDir(p)`    | `(s) → b`                 | `b`       | `true`, если путь — каталог                        |
| `store.size(p)`     | `(s) → i`                 | `i`       | Размер файла в байтах                              |
| `store.mkdir(p)`    | `(s) → b`                 | `b`       | Создание каталога (рекурсивно)                     |
| `store.glob(pat)`   | `(s) → array:s`           | `array:s` | Файлы по glob-шаблону                              |
| `store.zip(s, t)`   | `(s, s) → b`              | `b`       | Архивирование в zip                                |
| `store.unzip(z, d)` | `(s, s) → b`              | `b`       | Распаковка zip в каталог назначения                |

```xcx
store.write("log.txt", "First line");
store.append("log.txt", "\nSecond line");
s: content = store.read("log.txt");
if (store.exists("lock.pid")) then;
    >! "Already running";
end;
```

### env

| Метод           | Сигнатура      | Возврат    | Описание                                              |
|-----------------|----------------|------------|-------------------------------------------------------|
| `env.get(name)` | `(s) → s`      | `s`        | Значение переменной окружения; `halt.error`, если нет |
| `env.args()`    | `() → array:s` | `array:s`  | Аргументы командной строки программы как массив     |

```xcx
s: db_url = env.get("DATABASE_URL");

array:s: args = env.args();
for arg in args do;
    >! arg;
end;
```

### random

| Метод                             | Сигнатура           | Возврат | Описание                                                      |
|-----------------------------------|---------------------|---------|---------------------------------------------------------------|
| `random.choice from col`          | `(set/array) → T`   | `T`     | Случайный элемент из множества или массива                    |
| `random.int(min, max @step num)`  | `(i, i, @i) → i`    | `i`     | Случайное целое в `[min, max]`. Шаг по умолчанию: `1`.       |
| `random.float(min, max @step num)`| `(f, f, @f) → f`    | `f`     | Случайное вещественное в `[min, max]`. Шаг по умолчанию: `0.5`. |


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

### date (модуль)

```xcx
date: now = date.now();
```

---

## Система модулей

### include

Подключает код из другого файла в текущее пространство имён.

```xcx
include "utils.xcx";
include "math.xcx" as m;

m.PI;
m.sqrt(16.0);
```

Без псевдонима все символы доступны напрямую. С псевдонимом — с префиксом: `alias.symbol`.

Циклические зависимости обнаруживаются и отклоняются на этапе компиляции.
