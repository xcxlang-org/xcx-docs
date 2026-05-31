# Lexer XCX (Scanner) — v3.1

Le Lexer est responsable de la conversion du flux d'octets source brut en un flux de tokens discrets.

## Détails d'Implémentation

- **Fichier** : `src/lexer/scanner.rs`
- **Technique** : Analyse manuelle, eagerly, octet par octet sur `&[u8]`.
- **API** : Méthode unique `next_token(&mut self, interner: &mut Interner) -> Token`, appelée à la demande par le parseur (pas un itérateur).
- **Anticipation** : Anticipation d'un seul octet via `peek()`, de deux octets via `peek_next()` / `peek_at(offset)`.

## Structure Interne

```rust
pub struct Scanner<'a> {
    source:   &'a [u8],   // référence empruntée à la source originale — sans allocation
    pos:      usize,      // position en octets
    char_pos: usize,      // position en caractères Unicode (pour Span.col)
    line:     usize,
    col:      usize,
}
```

`Scanner` emprunte la chaîne source comme tranche d'octets. Il n'y a pas de conversion préalable en `Vec<char>` — toute l'analyse est au niveau octet. `char_pos` est incrémenté uniquement pour les octets qui ne sont pas des octets de continuation UTF-8 (`10xxxxxx`), de sorte que les comptages de caractères Unicode restent corrects pour les rapports de Span sans décoder chaque caractère.

## Types de Tokens

Les tokens sont définis dans `src/lexer/token.rs` comme l'enum `TokenKind`. Chaque `Token` porte un `Span { line, col, len }` pour les rapports d'erreur. `len` est mesuré en caractères Unicode (via des deltas `char_pos`), pas en octets.

Catégories clés :

| Catégorie | Exemples |
|---|---|
| Littéraux | `IntLiteral(i64)`, `FloatLiteral(f64)`, `StringLiteral(StringId)`, `True`, `False` |
| Mots-clés de type | `TypeI`, `TypeF`, `TypeS`, `TypeB`, `Array`, `Set`, `Map`, `Table`, `Json`, `Date`, `Fiber` |
| Mots-clés de type ensemble | `TypeSetN`, `TypeSetQ`, `TypeSetZ`, `TypeSetS`, `TypeSetB`, `TypeSetC` |
| Flux de contrôle | `If`, `Then`, `ElseIf`, `Else`, `End`, `While`, `Do`, `For`, `In`, `To`, `Break`, `Continue` |
| Fonctions/Fibres | `Func`, `Return`, `Fiber`, `Yield` |
| Opérateurs | `Plus`, `Minus`, `Star`, `Slash`, `Caret`, `PlusPlus`, `Has`, `And`, `Or`, `Not` |
| Opérateurs d'ensemble | `Union`, `Intersection`, `Difference`, `SymDifference` |
| Ponctuation spéciale | `GreaterBang` (`>!`), `GreaterQuestion` (`>?`), `DoubleColon` (`::`), `DoubleComma` (`,,`), `Bridge` (`<->`) |
| Builtins | `Net`, `Serve`, `Store`, `Halt`, `Terminal`, `Json`, `Date` |
| Spéciaux | `RawBlock(StringId)`, `AtStep`, `AtAuto`, `AtWait` |

## Fonctionnalités d'Analyse Spéciales

### Blocs Bruts
Délimités par `<<<` et `>>>`. Tout ce qui se trouve entre est capturé comme un seul token `RawBlock(StringId)`, utilisé pour JSON inline ou données de chaîne multi-lignes.

```
<<<
  { "key": "value" }
>>>
```

La détection utilise `self.source[self.pos..].starts_with(b"<<<")` — ouverte dans le bras principal de correspondance pour `<`. Le contenu brut est capturé octet par octet jusqu'à ce que `>>>` soit trouvé.

### Commentaires
XCX utilise `---` comme délimiteur de commentaire. La détection utilise `self.source[self.pos..].starts_with(b"---")` :
- **Sur une ligne** : `--- ceci est un commentaire` (contenu non-blanc sur la même ligne après `---`)
- **Multi-lignes** : `---` suivi uniquement d'espaces blancs jusqu'à la fin de ligne ouvre un bloc, fermé par `*---`

La fermeture multi-lignes est détectée par `self.source[self.pos..].starts_with(b"*---")`. Le scanner regarde en avant après avoir consommé `---` pour décider quel mode s'applique.

### Opérateurs d'Ensembles Unicode
Le scanner reconnaît les symboles Unicode via `starts_with` sur leurs séquences d'octets UTF-8 :
- `∪` → `TokenKind::Union`
- `∩` → `TokenKind::Intersection`
- `\` (barre oblique inverse ASCII) → `TokenKind::Difference`
- `⊕` → `TokenKind::SymDifference`

Pour les opérateurs Unicode multi-octets, `advance()` est appelé le nombre approprié de fois supplémentaires pour consommer les octets de continuation restants après l'octet de tête (`c >= 128`).

### Disambiguation Else/ElseIf
Le scanner regarde en avant après avoir reconnu `else` / `els` pour vérifier si le mot suivant est `if` — si c'est le cas, il combine les deux mots en un seul token `ElseIf`. La position sauvegardée (`after_ws_pos`, `after_ws_char_pos`, `after_ws_line`, `after_ws_col`) permet au scanner de revenir en arrière si le mot suivant n'est pas `if`.

### Directives `@`
Les tokens commençant par `@` sont analysés en consommant des octets ASCII alphabétiques et en correspondant au résultat :
- `@step` → `AtStep`
- `@auto` → `AtAuto`
- `@wait` → `AtWait`

Les séquences `@` inconnues produisent `TokenKind::Unknown('@')`.

### Point-Point (`..`) → `To`
Deux points consécutifs `..` sont analysés comme le token `To` (utilisé dans les expressions de plage), distinct d'un seul `.` (`Dot`).

### Double-Virgule (`,,`) → `DoubleComma`
Deux virgules consécutives `,,` sont analysées comme `DoubleComma` (utilisé dans les littéraux de plage d'ensemble : `set:N { 1,,10 }`).

### Analyse des Identifiants
`identifier()` capture une suite contiguë d'octets ASCII alphanumériques, des underscores et des octets `>= 128` (pour inclure les identifiants UTF-8). La plage d'octets capturée est convertie via `std::str::from_utf8` et mise en minuscules pour la correspondance de mots-clés. Les correspondances sensibles à la casse sont vérifiées en premier (p.ex., `"N"`, `"Q"`, `"Z"` pour les types d'ensemble, `"UNION"`, `"HAS"`, `"AND"` pour les variantes de mots-clés en majuscules). Les identifiants non-mots-clés sont passés à `Interner::intern()`.

### Analyse des Nombres
`number()` accumule des octets de chiffres ASCII. Si un `.` suivi d'un chiffre est rencontré, le token devient un `FloatLiteral`. La plage d'octets est convertie via `str::from_utf8` et analysée avec `.parse()`.

### Analyse des Chaînes
`string()` traite les séquences d'échappement (`\n`, `\t`, `\r`, `\"`, `\\`, octal `\NNN`, hex `\xHH`) octet par octet, construisant un `Vec<u8>` qui est ensuite converti via `String::from_utf8`. Le résultat est interné.

## Allocation de Chaînes

Tous les identifiants et littéraux de chaînes sont passés par `Interner::intern()`, retournant un `StringId (u32)`. La `String` brute est stockée une fois dans le `Vec<String>` interne de l'internement ; le reste du pipeline travaille avec des IDs numériques, éliminant les comparaisons sur le tas pendant la vérification des types et la compilation.