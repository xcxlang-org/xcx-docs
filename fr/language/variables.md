# Variables et Constantes XCX 3.1

## Déclarations de Variables

Les variables sont déclarées avec la syntaxe `type: nom = valeur;`.

```xcx
i: age = 25;
f: pi = 3.14159;
s: name = "XCX";
b: is_ready = true;
```

Les variables peuvent également être déclarées sans valeur initiale ; elles prennent alors la valeur par défaut de leur type (ex. `0` pour `i`, `""` pour `s`).

## Réassignation

Une fois déclarée, une variable peut être réassignée avec l'opérateur `=`.

```xcx
i: count = 0;
count = count + 1;
```

## Constantes

Les constantes sont déclarées avec le mot-clé `const`. Elles doivent être initialisées à la déclaration et ne peuvent pas être modifiées par la suite.

```xcx
const i: MAX_CONNECTIONS = 1024;
const s: VERSION = "2.0.0";
```

## Pas d'Ombrage de Variable

XCX ne supporte **pas** l'ombrage de variable. Déclarer une variable avec le même nom dans n'importe quelle portée — y compris un bloc imbriqué — est une **erreur à la compilation** (`RedefinedVariable`).

```xcx
i: x = 10;
if (true) then;
    i: x = 20;   --- COMPILE ERROR: RedefinedVariable
end;
```

Pour modifier une valeur à l'intérieur d'un bloc, utilisez la réassignation plutôt que la redéclaration :

```xcx
i: x = 10;
if (true) then;
    x = 20;      --- OK: reassignment to existing variable
    >! x;        --- 20
end;
>! x;            --- 20 (the global variable was modified)
```