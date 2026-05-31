# E/S et Commandes Système XCX 3.1

## Sortie Console (`>!`)

L'opérateur `>!` affiche des valeurs sur `stdout`, suivi d'un saut de ligne.

```xcx
>! "Hello";
>! 42;
>! "Path: " + path;
```

Les **séquences d'échappement** comme `\n` (saut de ligne), `\t` (tabulation) et `\r` (retour chariot) sont supportées.

## Entrée Utilisateur (`>?`)

L'opérateur `>?` lit une ligne depuis `stdin` et tente de l'analyser dans la variable cible.

```xcx
i: age;
>! "Enter age:";
>? age;
```

## Délai (`@wait`)

Met en pause l'exécution de la VM pendant un nombre de millisecondes spécifié.

```xcx
@wait 1000; --- Waits for 1 second
```

> `@wait` est une opération **bloquante**. Elle ne cède pas la fibre.

## Entrée Brute (`input`)

Le module `input` permet de lire des touches individuelles sans attendre Entrée.

### Méthodes

| Appel               | Retour  | Description                                         |
|---------------------|---------|-----------------------------------------------------|
| `input.key()`       | `s`     | Retourne la touche si disponible, `""` sinon        |
| `input.key() @wait` | `s`     | Attend une touche et la retourne                    |
| `input.ready()`     | `b`     | `true` si une touche attend dans le tampon          |

### Constantes de Touches

| Valeur          | Touche          |
|-----------------|-----------------|
| `"UP"`          | Flèche Haut     |
| `"DOWN"`        | Flèche Bas      |
| `"LEFT"`        | Flèche Gauche   |
| `"RIGHT"`       | Flèche Droite   |
| `"ENTER"`       | Entrée          |
| `"ESC"`         | Échap           |
| `"BACKSPACE"`   | Retour arrière  |
| `"TAB"`         | Tab             |
| `"F1"` ... `"F12"` | F1–F12       |
| `"KEY_CTRL_C"`  | Ctrl+C          |
| `"KEY_CTRL_Z"`  | Ctrl+Z          |
| `"KEY_CTRL_S"`  | Ctrl+S          |

Les caractères normaux sont retournés directement : `"a"`, `"Z"`, `"5"`, `" "`.

### Exemple

```xcx
s: k = input.key();
if (k == "UP") then;
    y = y - 1;
end;

--- wait for specific key
s: confirm = input.key() @wait;
if (confirm == "q") then;
    return;
end;
```

## Commandes Terminal (`.terminal`)

Interagissez directement avec l'environnement système ou le processus en cours.

| Directive              | Description                                          |
|------------------------|------------------------------------------------------|
| `.terminal !clear`     | Efface l'écran                                       |
| `.terminal !exit`      | Termine le processus VM                              |
| `.terminal !run s`     | Exécute un autre fichier XCX dans un nouveau processus |
| `.terminal !raw`       | Mode brut — pas d'écho, pas de tampon                |
| `.terminal !normal`    | Restaure le mode terminal normal                     |
| `.terminal !cursor on` | Affiche le curseur                                   |
| `.terminal !cursor off`| Cache le curseur                                     |
| `.terminal !move x y`  | Déplace le curseur à la position `x`, `y` (`i`)      |
| `.terminal !write expr`| Affiche `expr` sans saut de ligne final              |

### Exemple — Boucle de Jeu

```xcx
.terminal !raw;
.terminal !cursor off;
.terminal !clear;

while (true) do;
    s: k = input.key();
    if (k == "ESC") then; break; end;
    if (k == "UP") then; y = y - 1; end;
    .terminal !move x y;
    .terminal !write "@";
    @wait 16;
end;

.terminal !cursor on;
.terminal !normal;
```

## Gestion des Erreurs et Contraintes

- **`input.key()` en mode normal** : Retourne `""` et affiche un avertissement.
- **`@wait` sur `input.ready()`** : Erreur à la compilation.
- **Directives `.terminal`** : Ne sont pas des expressions et ne peuvent pas être assignées à des variables.
- **`.terminal !move`** : Les arguments doivent être des entiers (`i`).
- **Disponibilité du Terminal** : La VM s'arrêtera avec une erreur fatale si les handles de console sont redirigés ou indisponibles.