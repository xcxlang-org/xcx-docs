# XCX 3.1 — функции и файберы

> [!IMPORTANT]
> **Правило глобальной области**: функции и файберы объявляются только на **верхнем уровне** файла. Их нельзя вкладывать в блоки `if`/`while`/`for` или в другие функции/файберы так, чтобы это влияло на область видимости — все имена регистрируются глобально при компиляции.

## Функции

### Стили определения

Параметры следуют шаблону `type: name` (например, `i: count`).

```xcx
--- No parameters, no return value
func hello() {
    >! "Hello!";
};

--- No parameters, with return value
func get_pi(-> f) {
    return 3.14159;
};

--- With parameters, no return value
func greet(s: who) {
    >! "Hello, " + who + "!";
};

--- With parameters and return value
func add(i: x, i: y -> i) {
    return x + y;
};
```

### Вызов функций

```xcx
hello();
f: pi = get_pi();
greet("Alice");
i: result = add(3, 4);
```

### Рекурсия

Максимальная глубина: **800 кадров**. Превышение вызывает `halt.error`.

```xcx
func factorial(i: n -> i) {
    if (n <= 1) then;
        return 1;
    else;
        return n * factorial(n - 1);
    end;
};
>! factorial(5);   --- 120
```

---

## Файберы (корутины)

Файбер может приостанавливать выполнение через `yield` и возобновляться. Есть два варианта: **типизированный** (yield возвращает значение) и **пустой** (yield без значения).

### Определение

```xcx
--- Typed Fiber: each yield returns a value of type T
fiber generator(i: max -> i) {
    for i in 1 to max do;
        yield i;
    end;
    return 0;
};

--- Void Fiber: yield without a value
fiber logger(s: msg) {
    if (msg == "") then; return; end;
    >! msg;
    yield;
};
```

### Создание экземпляра

```xcx
fiber:i: gen  = generator(10);   --- typed
fiber:  log   = logger("init");  --- void
```

### Методы файбера

| Метод       | Для        | Возврат | Описание                                      |
|-------------|------------|---------|-----------------------------------------------|
| `.next()`   | `fiber:T`  | `T`     | Возобновление; следующее переданное значение  |
| `.run()`    | `fiber:`   | `b`     | Один шаг пустого файбера                      |
| `.isDone()` | оба        | `b`     | `true`, когда файбер завершён                 |
| `.close()`  | оба        | `b`     | Принудительное завершение файбера             |

Вызов `.next()` при `isDone() == true` вызывает `halt.alert` (R306).

### yield

```xcx
--- In a typed fiber
yield true;
yield score + 10;

--- In a void fiber
yield;
```

### return в файберах

1. **Типизированный файбер** (`fiber ... -> T`): требуется `return expr;` со значением типа `T`. Голый `return;` без значения — ошибка компиляции (S210).
2. **Пустой файбер**: для досрочного выхода используется `return;`.

```xcx
fiber:i: gen(i: max -> i) {
    for i in 1 to max do;
        if (i == 3) then;
            return 99;   --- Valid: terminates and returns 99
        end;
        yield i;
    end;
    return 0;            --- Required at end of typed fiber
};

fiber logger(s: msg) {
    if (msg == "") then;
        return;          --- Valid for void fiber
    end;
    >! msg;
};
```

### Цикл for по файберу

```xcx
fiber:i: f = generator(5);
for val in f do;
    >! val;
end;
```

Эквивалентно `while (!f.isDone()) { val = f.next(); ... }`. `break` в цикле автоматически вызывает `.close()`.

### Файбер как параметр функции

```xcx
func run_checks(fiber:b: f) {
    while (f.isDone() == false) do;
        b: result = f.next();
        if (result == false) then;
            halt.alert >! "Step failed";
            return;
        end;
    end;
};

fiber:b: checks = login_pipeline(user, pass);
run_checks(checks);
```

### yield from (делегирование)

Делегирует выполнение под-файберу. Компилятор разворачивает это в цикл, передающий каждое значение из под-файбера.

```xcx
fiber count(i: n -> i) {
    i: x = 0;
    while (x < n) do;
        yield x;
        x = x + 1;
    end;
    return 0;
};

fiber delegator(-> i) {
    fiber:i: sub = count(3);
    yield from sub;   --- yields: 0, 1, 2
    yield 100;        --- then yields: 100
    return 0;
};
```

### Ручное делегирование

```xcx
fiber pipeline(s: user, s: pass -> b) {
    fiber:b: step1 = check_user(user);
    yield step1.next();

    fiber:b: step2 = check_pass(user, pass);
    yield step2.next();

    return false;
};
```
