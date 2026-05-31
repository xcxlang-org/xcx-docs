# Fonctions et Fibres XCX 3.1

> [!IMPORTANT]
> **Règle de Portée Globale** : Les fonctions et fibres doivent être déclarées au **niveau supérieur** d'un fichier. Elles ne peuvent pas être imbriquées dans des blocs `if`/`while`/`for` ou d'autres fonctions/fibres d'une manière qui affecte la portée — tous les noms de fonctions et fibres sont enregistrés globalement lors de la compilation.

## Fonctions

### Styles de Définition

Les paramètres suivent le motif `type: nom` (ex. `i: count`).

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

### Appel de Fonctions

```xcx
hello();
f: pi = get_pi();
greet("Alice");
i: result = add(3, 4);
```

### Récursion

Profondeur maximale : **800 frames**. Dépasser cette limite déclenche `halt.error`.

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

## Fibres (Coroutines)

Les fibres peuvent suspendre leur exécution avec `yield` et être reprises. Il existe deux variantes : **typée** (yield renvoie une valeur) et **void** (yield sans valeur).

### Définition

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

### Instanciation

```xcx
fiber:i: gen  = generator(10);   --- typed
fiber:  log   = logger("init");  --- void
```

### Méthodes de Fibre

| Méthode     | S'applique à | Retour | Description                                   |
|-------------|--------------|--------|-----------------------------------------------|
| `.next()`   | `fiber:T`    | `T`    | Reprend ; retourne la prochaine valeur cédée   |
| `.run()`    | `fiber:`     | `b`    | Reprend la fibre void d'un pas                |
| `.isDone()` | les deux     | `b`    | `true` quand la fibre est terminée            |
| `.close()`  | les deux     | `b`    | Termine la fibre de force                     |

Appeler `.next()` quand `isDone() == true` déclenche `halt.alert` (R306).

### yield

```xcx
--- In a typed fiber
yield true;
yield score + 10;

--- In a void fiber
yield;
```

### return dans les Fibres

1. **Fibre typée** (`fiber ... -> T`) : Requiert `return expr;` avec une valeur de type `T`. Un `return;` nu sans valeur est une erreur à la compilation (S210).
2. **Fibre void** : Utilise `return;` pour quitter prématurément.

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

### Boucle for sur une Fibre

```xcx
fiber:i: f = generator(5);
for val in f do;
    >! val;
end;
```

Équivalent à `while (!f.isDone()) { val = f.next(); ... }`. `break` à l'intérieur de la boucle appelle automatiquement `.close()`.

### Fibre comme Paramètre de Fonction

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

### yield from (Délégation)

Délègue l'exécution à une sous-fibre. Le compilateur désugarise cela en une boucle qui cède chaque valeur de la sous-fibre.

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

### Délégation Manuelle

```xcx
fiber pipeline(s: user, s: pass -> b) {
    fiber:b: step1 = check_user(user);
    yield step1.next();

    fiber:b: step2 = check_pass(user, pass);
    yield step2.next();

    return false;
};
```