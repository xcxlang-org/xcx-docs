# Date et Heure XCX 3.1

## Création

Les dates sont créées avec la fonction `date()` ou `date.now()`.

```xcx
date: d1 = date("2024-12-25");               --- YYYY-MM-DD
date: d2 = date("25/12/2024", "DD/MM/YYYY"); --- Custom format
date: now = date.now();                      --- Current system time
```

## Propriétés

Les objets date exposent des champs entiers en lecture seule :

- `.year`
- `.month` (1-12)
- `.day` (1-31)
- `.hour` (0-23)
- `.minute` (0-59)
- `.second` (0-59)

## Arithmétique et Comparaison

Les dates supportent l'addition/soustraction de jours et la comparaison avec d'autres dates.

```xcx
date: tomorrow = some_date + 1;
date: yesterday = some_date - 1;
i: days_between = christmas - halloween;  --- Result in integer days
b: is_before = christmas < new_year;
```

## Formatage

```xcx
s: s1 = now.format();                    --- Default: "YYYY-MM-DD HH:mm:ss"
s: s2 = now.format("DD/MM/YYYY HH:mm");  --- Custom output
```

**Jetons :** `YYYY` (Année), `MM` (Mois 01-12), `DD` (Jour 01-31), `HH` (Heure 00-23), `mm` (Minute), `ss` (Seconde), `SSS` ou `ms` (Millisecondes), `M` et `D` (sans zéro initial).