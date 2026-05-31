# Lexer XCX (Scanner) — v3.0

Lexer odpowiada za konwersję surowego strumienia bajtów źródła na strumień dyskretnych tokenów.

## Szczegóły implementacji

- **Plik**: `src/lexer/scanner.rs`
- **Technika**: Ręczne, eager skanowanie bajt po bajcie na `&[u8]`.
- **API**: Jedna metoda `next_token(&mut self, interner: &mut Interner) -> Token`, wywoływana na żądanie przez parser (nie iterator).
- **Lookahead**: Jednobajtowy lookahead przez `peek()`, dwubajtowy przez `peek_next()` / `peek_at(offset)`.

## Struktura wewnętrzna

```rust
pub struct Scanner<'a> {
    source:   &'a [u8],   // borrowed reference to original source — no allocation
    pos:      usize,      // byte position
    char_pos: usize,      // Unicode character position (for Span.col)
    line:     usize,
    col:      usize,
}
```

`Scanner` pożycza łańcuch źródłowy jako wycinek bajtów. Nie ma up-front konwersji do `Vec<char>` — całe skanowanie jest na poziomie bajtów. `char_pos` jest zwiększany tylko dla bajtów, które nie są bajtami kontynuacji UTF-8 (`10xxxxxx`), więc liczniki znaków Unicode pozostają poprawne dla raportowania Span bez dekodowania każdego znaku.

## Typy tokenów

Tokeny zdefiniowane są w `src/lexer/token.rs` jako enum `TokenKind`. Każdy `Token` niesie `Span { line, col, len }` do raportowania błędów. `len` mierzone jest w znakach Unicode (przez delty `char_pos`), nie w bajtach.

Kluczowe kategorie:

| Kategoria | Przykłady |
|---|---|
| Literały | `IntLiteral(i64)`, `FloatLiteral(f64)`, `StringLiteral(StringId)`, `True`, `False` |
| Słowa kluczowe typów | `TypeI`, `TypeF`, `TypeS`, `TypeB`, `Array`, `Set`, `Map`, `Table`, `Json`, `Date`, `Fiber` |
| Słowa kluczowe typów zbiorów | `TypeSetN`, `TypeSetQ`, `TypeSetZ`, `TypeSetS`, `TypeSetB`, `TypeSetC` |
| Przepływ sterowania | `If`, `Then`, `ElseIf`, `Else`, `End`, `While`, `Do`, `For`, `In`, `To`, `Break`, `Continue` |
| Funkcje/Fibery | `Func`, `Return`, `Fiber`, `Yield` |
| Operatory | `Plus`, `Minus`, `Star`, `Slash`, `Caret`, `PlusPlus`, `Has`, `And`, `Or`, `Not` |
| Operatory zbiorów | `Union`, `Intersection`, `Difference`, `SymDifference` |
| Specjalna interpunkcja | `GreaterBang` (`>!`), `GreaterQuestion` (`>?`), `DoubleColon` (`::`), `DoubleComma` (`,,`), `Bridge` (`<->`) |
| Wbudowane | `Net`, `Serve`, `Store`, `Halt`, `Terminal`, `Json`, `Date` |
| Specjalne | `RawBlock(StringId)`, `AtStep`, `AtAuto`, `AtWait` |

## Specjalne funkcje skanowania

### Bloki raw
Ograniczone przez `<<<` i `>>>`. Wszystko między nimi trafia do jednego tokenu `RawBlock(StringId)`, używanego dla inline JSON lub wieloliniowych danych łańcuchowych.

```
<<<
  { "key": "value" }
>>>
```

Wykrywanie używa `self.source[self.pos..].starts_with(b"<<<")` — otwierane w ramach rozróżniania `<` w głównej gałęzi match. Surowa treść jest zbierana bajt po bajcie aż do `>>>`.

### Komentarze
XCX używa `---` jako ogranicznika komentarzy. Wykrywanie: `self.source[self.pos..].starts_with(b"---")`:
- **Jednoliniowy**: `--- this is a comment` (niebiała treść w tej samej linii po `---`)
- **Wieloliniowy**: `---` a po nim tylko białe znaki do końca linii otwiera blok, zamykany przez `*---`

Zamknięcie wieloliniowe: `self.source[self.pos..].starts_with(b"*---")`. Skaner zagląda do przodu po skonsumowaniu `---`, aby ustalić tryb.

### Operatory zbiorów Unicode
Skaner rozpoznaje symbole Unicode przez `starts_with` na sekwencjach bajtów UTF-8:
- `∪` → `TokenKind::Union`
- `∩` → `TokenKind::Intersection`
- `\` (ASCII backslash) → `TokenKind::Difference`
- `⊕` → `TokenKind::SymDifference`

Dla wielobajtowych operatorów Unicode `advance()` wywoływane jest odpowiednią liczbę dodatkowych razy, aby skonsumować pozostałe bajty kontynuacji po bajcie wiodącym (`c >= 128`).

### Rozróżnianie Else/ElseIf
Skaner zagląda do przodu po rozpoznaniu `else` / `els`, sprawdzając czy następne słowo to `if` — jeśli tak, scala oba słowa w jeden token `ElseIf`. Zapisana pozycja (`after_ws_pos`, `after_ws_char_pos`, `after_ws_line`, `after_ws_col`) pozwala na cofnięcie, jeśli następne słowo nie jest `if`.

### Dyrektywy `@`
Tokeny zaczynające się od `@` skanowane są przez zbieranie bajtów alfabetu ASCII i dopasowanie wyniku:
- `@step` → `AtStep`
- `@auto` → `AtAuto`
- `@wait` → `AtWait`

Nieznane sekwencje `@` produkują `TokenKind::Unknown('@')`.

### Kropka-kropka (`..`) → `To`
Dwie kolejne kropki `..` skanowane są jako token `To` (w wyrażeniach zakresu), odróżnione od pojedynczej `.` (`Dot`).

### Podwójny przecinek (`,,`) → `DoubleComma`
Dwa kolejne przecinki `,,` skanowane są jako `DoubleComma` (w literałach zakresu zbiorów: `set:N { 1,,10 }`).

### Skanowanie identyfikatorów
`identifier()` zbiera ciągły bieg bajtów alfanumerycznych ASCII, podkreśleń i bajtów `>= 128` (identyfikatory UTF-8). Zakres bajtów konwertowany jest przez `std::str::from_utf8` i sprowadzany do małych liter przed dopasowaniem słów kluczowych. Najpierw sprawdzane są dopasowania wrażliwe na wielkość liter (np. `"N"`, `"Q"`, `"Z"` dla typów zbiorów, `"UNION"`, `"HAS"`, `"AND"` dla wariantów wielkich liter). Identyfikatory niebędące słowami kluczowymi trafiają do `Interner::intern()`.

### Skanowanie liczb
`number()` akumuluje bajty cyfr ASCII. Jeśli `.` z następną cyfrą, token staje się `FloatLiteral`. Zakres bajtów konwertowany jest przez `str::from_utf8` i parsowany przez `.parse()`.

### Skanowanie łańcuchów
`string()` przetwarza sekwencje escape (`\n`, `\t`, `\r`, `\"`, `\\`, ósemkowe `\NNN`, szesnastkowe `\xHH`) bajt po bajcie, budując `Vec<u8>` konwertowany potem przez `String::from_utf8`. Wynik jest internowany.

## Alokacja łańcuchów

Wszystkie identyfikatory i literały łańcuchowe przechodzą przez `Interner::intern()`, zwracając `StringId (u32)`. Surowy `String` przechowywany jest raz w wewnętrznym `Vec<String>` internera; reszta potoku pracuje na numerycznych ID, eliminując porównania na stercie podczas sprawdzania typów i kompilacji.
