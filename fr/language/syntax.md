# Bases de la Syntaxe XCX 3.1

## Commentaires

```xcx
--- Single-line comment
i: price = 100;  --- inline comment

---
Multi-line
comment block.
*---
```

## Identifiants

Les identifiants sont **sensibles à la casse** et doivent correspondre au motif `[a-zA-Z][a-zA-Z0-9_]*`.

| Valide        | Invalide     | Raison                              |
|---------------|-------------|--------------------------------------|
| `userData`    | `1stUser`    | Ne peut pas commencer par un chiffre |
| `user_data`   | `user-data`  | Le trait d'union est un opérateur moins |
| `counter1`    | `user name`  | Les espaces ne sont pas autorisés    |

Les mots-clés (`if`, `func`, `fiber`, `i`, `f`, `s`, `b`, etc.) sont réservés.

## Terminaison des Instructions

Les instructions se terminent par un point-virgule `;`. Cela s'applique aux déclarations de variables, aux appels de fonctions, aux affectations et aux directives.

```xcx
i: x = 10;
>! "Hello";
```

## Blocs

Les blocs sont ouverts par un mot-clé (ex. `then`, `do`, `{`) et fermés par `end;`.

```xcx
if (x > 0) then;
    >! "Positive";
end;
```