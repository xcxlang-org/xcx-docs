# Structures de Contrôle XCX 3.1

## Instructions Conditionnelles (if/elseif/else)

```xcx
if (condition) then;
    --- block
elseif (other_condition) then;
    --- block
else;
    --- block
end;
```

**Alias :**

| Mot-clé   | Alias          |
|-----------|----------------|
| `elseif`  | `elif`, `elf`  |
| `else`    | `els`          |

Toutes les formes sont équivalentes et peuvent être mélangées :

```xcx
i: score = 75;
if (score >= 90) then;
    >! "A";
elif (score >= 75) then;
    >! "B";
elf (score >= 60) then;
    >! "C";
els;
    >! "F";
end;
```

---

## Boucles

### For à Plage Numérique

La plage est **inclusive** des deux côtés.

```xcx
for i in 1 to 3 do;
    >! i;
end;
--- prints: 1, 2, 3
```

### For avec Pas (`@step`)

```xcx
for j in 0 to 6 @step 2 do;
    >! j;
end;
--- prints: 0, 2, 4, 6
```

### Itération sur une Collection

Fonctionne sur les tableaux, ensembles et fibres. La variable de boucle reçoit l'**élément**, pas l'index.

```xcx
--- Array
array:i: nums {10, 20, 30};
for el in nums do;
    >! el;
end;

--- Set
set:N: primes {2, 3, 5, 7, 11};
for p in primes do;
    >! p;
end;

--- Fiber
fiber:i: f = gen(5);
for val in f do;
    >! val;
end;
```

### Boucle While

```xcx
i: cnt = 0;
while (cnt < 3) do;
    cnt = cnt + 1;
    >! cnt;
end;
```

### Break et Continue

`break` quitte la boucle en cours. `continue` passe à l'itération suivante. Les deux n'affectent que la **boucle immédiatement englobante**.

```xcx
for n in 1 to 5 do;
    if (n % 2 == 0) then; continue; end;
    if (n == 5) then; break; end;
    >! n;
end;
```

> [!NOTE]
> `break` à l'intérieur d'une boucle `for` sur une fibre appelle automatiquement `.close()` sur cette fibre.