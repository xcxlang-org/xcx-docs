# Opérateurs XCX 3.1

## Opérateurs Arithmétiques

| Opérateur | Description                          | Exemple    | Résultat |
|-----------|--------------------------------------|------------|----------|
| `+`       | Addition                             | `10 + 3`   | `13`     |
| `-`       | Soustraction / Négation unaire       | `10 - 3`   | `7`      |
| `*`       | Multiplication                       | `10 * 3`   | `30`     |
| `/`       | Division (tronque pour `i`)          | `10 / 3`   | `3`      |
| `%`       | Modulo (reste)                       | `10 % 3`   | `1`      |
| `^`       | Exponentiation                       | `2 ^ 10`   | `1024`   |
| `++`      | Concaténation de chiffres entiers    | `48 ++ 77` | `4877`   |

## Opérateurs de Chaîne

| Opérateur | Description              | Exemple                  | Résultat         |
|-----------|--------------------------|--------------------------|------------------|
| `+`       | Concaténation            | `"Hi " + "User"`         | `"Hi User"`      |
| `HAS`     | Recherche de sous-chaîne | `"Search" HAS "ear"`     | `true`           |

> [!TIP]
> La concaténation de chaînes convertit automatiquement les nombres et les booléens en chaînes (ex. `"Value: " + 42` → `"Value: 42"`).

## Opérateurs Logiques

| Opérateur | Alias | Description |
|-----------|-------|-------------|
| `AND`     | `&&`  | ET logique  |
| `OR`      | `||`  | OU logique  |
| `NOT`     | `!!`  | NON logique |

## Opérateurs de Comparaison

Utilisés pour comparer des valeurs du même type.

| Opérateur | Description              |
|-----------|--------------------------|
| `==`      | Égal à                   |
| `!=`      | Différent de             |
| `<`       | Inférieur à              |
| `>`       | Supérieur à              |
| `<=`      | Inférieur ou égal à      |
| `>=`      | Supérieur ou égal à      |

## Opérateurs d'Ensemble

| Mot                    | Symbole | Description           |
|------------------------|---------|-----------------------|
| `UNION`                | `∪`     | Union d'ensembles     |
| `INTERSECTION`         | `∩`     | Intersection          |
| `DIFFERENCE`           | `\`     | Différence            |
| `SYMMETRIC_DIFFERENCE` | `⊕`     | Différence symétrique |

## Précédence des Opérateurs (du plus élevé au plus bas)

1. Appels de fonctions, appels de méthodes, indexation `[]`
2. Négation unaire `-`, `!!` (NOT)
3. Exponentiation `^`
4. Multiplication `*`, Division `/`, Modulo `%`
5. Opérations d'ensemble `UNION`, `INTERSECTION`, `DIFFERENCE`, `SYM_DIFF`
6. Addition `+`, Soustraction `-`, Concaténation `++`
7. Comparaisons `HAS`, `>`, `<`, `>=`, `<=`
8. Égalité `==`, `!=`
9. Logique `AND`, `&&`
10. Logique `OR`, `||`