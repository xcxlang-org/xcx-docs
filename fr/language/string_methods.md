# Méthodes de Chaîne XCX 3.1

Les chaînes dans XCX sont immuables. Les méthodes retournent une **nouvelle chaîne** et ne modifient pas l'originale.

## Propriétés

- `.length` : Retourne le nombre de points de code Unicode dans la chaîne (ex. `"zażółć".length` est 6). **Utilisé sans parenthèses.**

## Séquences d'Échappement

Les littéraux de chaîne supportent les séquences d'échappement standard :

| Séquence | Effet |
|----------|-------|
| `\n`     | Saut de ligne |
| `\t`     | Tabulation horizontale |
| `\r`     | Retour chariot |
| `\"`     | Guillemet double |
| `\\`     | Barre oblique inverse |
| `\xNN`   | Caractère hexadécimal (ex. `\x1b`) |
| `\NNN`   | Caractère octal (ex. `\033`) |

## Méthodes

| Méthode              | Signature    | Description                                       |
|----------------------|--------------|---------------------------------------------------|
| `.upper()`           | `() → s`     | Convertit tous les caractères en majuscules.       |
| `.lower()`           | `() → s`     | Convertit tous les caractères en minuscules.       |
| `.trim()`            | `() → s`     | Supprime les espaces en début/fin.                 |
| `.replace(f, t)`     | `(s, s) → s` | Remplace toutes les occurrences de `f` par `t`.    |
| `.slice(s, e)`       | `(i, i) → s` | Retourne la sous-chaîne de l'index `s` jusqu'à `e`.|
| `.indexOf(s)`        | `(s) → i`    | Retourne l'index de la première occurrence, ou -1. |
| `.lastIndexOf(s)`    | `(s) → i`    | Retourne l'index de la dernière occurrence, ou -1. |
| `.startsWith(s)`     | `(s) → b`    | `true` si la chaîne commence par `s`.              |
| `.endsWith(s)`       | `(s) → b`    | `true` si la chaîne se termine par `s`.            |
| `.toInt()`           | `() → i`     | Analyse la chaîne en entier ; `halt.error` si échec.|
| `.toFloat()`         | `() → f`     | Analyse la chaîne en flottant ; `halt.error` si échec.|
| `.split(s)`          | `(s) → array:s` | Divise par le séparateur `s` ; retourne un tableau de chaînes. |

## Exemples

```xcx
s: raw = "  XCX-Language  ";
s: clean = raw.trim().lower().replace("-", "_"); --- "xcx_language"

i: start = "Programming".indexOf("gram");       --- 3
s: part = "Programming".slice(0, 4);            --- "Prog"

i: age = "25".toInt();
array:s: parts = "a,b,c".split(",");           --- {"a", "b", "c"}
```