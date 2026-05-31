# XCX 3.1 Funkcje i włókna

> [!IMPORTANT]
> **Reguła zakresu globalnego**: Funkcje i włókna muszą być deklarowane na **najwyższym poziomie** pliku. Nie mogą być zagnieżdżone w blokach `if`/`while`/`for` ani w innych funkcjach/włóknach w sposób wpływający na zakres — wszystkie nazwy funkcji i włókien są rejestrowane globalnie podczas kompilacji.

## Funkcje

### Style definicji

Parametry używają wzorca `type: name` (np. `i: count`).

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

### Wywoływanie funkcji

```xcx
hello();
f: pi = get_pi();
greet("Alice");
i: result = add(3, 4);
```

### Rekurencja

Maksymalna głębokość: **800 ramek**. Przekroczenie wywołuje `halt.error`.

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

## Włókna (korutyny)

Włókna mogą wstrzymywać wykonanie za pomocą `yield` i być wznawiane. Istnieją dwa warianty: **typowane** (`yield` zwraca wartość) i **void** (`yield` bez wartości).

### Definicja

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

### Instancjonowanie

```xcx
fiber:i: gen  = generator(10);   --- typed
fiber:  log   = logger("init");  --- void
```

### Metody włókien

| Metoda      | Działa na  | Zwraca | Opis                                        |
|-------------|------------|--------|---------------------------------------------|
| `.next()`   | `fiber:T`  | `T`    | Wznawia; zwraca następną wyprodukowaną wartość |
| `.run()`    | `fiber:`   | `b`    | Wznawia włókno void o jeden krok            |
| `.isDone()` | oba        | `b`    | `true`, gdy włókno zakończyło działanie     |
| `.close()`  | oba        | `b`    | Wymusza zakończenie włókna                  |

Wywołanie `.next()`, gdy `isDone() == true`, wywołuje `halt.alert` (R306).

### yield

```xcx
--- In a typed fiber
yield true;
yield score + 10;

--- In a void fiber
yield;
```

### return w włóknach

1. **Włókno typowane** (`fiber ... -> T`): Wymaga `return expr;` z wartością typu `T`. Gołe `return;` bez wartości to błąd kompilacji (S210).
2. **Włókno void**: Używa `return;` do wcześniejszego wyjścia.

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

### Pętla for po włóknie

```xcx
fiber:i: f = generator(5);
for val in f do;
    >! val;
end;
```

Równoważne z `while (!f.isDone()) { val = f.next(); ... }`. `break` w pętli automatycznie wywołuje `.close()`.

### Włókno jako parametr funkcji

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

### yield from (delegacja)

Deleguje wykonanie do podwłókna. Kompilator rozwija to do pętli, która produkuje każdą wartość z podwłókna.

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

### Ręczna delegacja

```xcx
fiber pipeline(s: user, s: pass -> b) {
    fiber:b: step1 = check_user(user);
    yield step1.next();

    fiber:b: step2 = check_pass(user, pass);
    yield step2.next();

    return false;
};
```
