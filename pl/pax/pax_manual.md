# Podręcznik menedżera pakietów PAX

PAX to oficjalny menedżer pakietów dla XCX, zintegrowany bezpośrednio z binarią `xcx`. Zarządza zależnościami, szkieletem projektu i automatyzacją buildów.

## Konfiguracja projektu: `project.pax`

Każdy projekt PAX musi mieć plik `project.pax` w katalogu głównym. Używa niestandardowego formatu deklaratywnego.

```pax
---
PAX Project Configuration
*---
/
    name :: "my_project",
    deps :: [
        "user/repo",
        "https://example.com/lib.xcx"
    ]
/
```

- **name**: Logiczna nazwa projektu.
- **deps**: Lista zależności. Obsługuje skróty GitHub (`user/repo`) i bezpośrednie URL-e.

## Referencja poleceń

PAX jest wywoływany przez `xcx pax <command>`.

| Polecenie                  | Opis                                          |
|----------------------------|-----------------------------------------------|
| `xcx pax new <name>`       | Generuje nową strukturę projektu.             |
| `xcx pax clone <package>`  | Pobiera opublikowany pakiet jako lokalny projekt. |
| `xcx pax install`          | Pobiera zależności do katalogu `lib/`.        |
| `xcx pax add <dep>`        | Dodaje zależność i instaluje ją natychmiast.  |
| `xcx pax remove <name>`    | Usuwa zależność z `project.pax`.              |
| `xcx pax search <query>`   | Przeszukuje rejestr dostępnych pakietów.      |
| `xcx pax run [path]`       | Uruchamia projekt (wejście: `src/main.xcx`).  |

### `xcx pax clone <package>`

Pobiera opublikowany pakiet z rejestru PAX do nowego lokalnego katalogu, gotowego do uruchomienia lub modyfikacji.

```sh
xcx pax clone snake_game
xcx pax clone beta/snake_game
```

- Argument to nazwa pakietu z rejestru (opcjonalnie z prefiksem `author/`).
- Tworzy katalog nazwany od pakietu w bieżącym katalogu roboczym.
- Kopiuje pełną strukturę projektu: `project.pax`, `src/` i wszystkie inne pliki zadeklarowane w `files`.
- **Nie** instaluje zależności automatycznie — po sklonowaniu uruchom `xcx pax install` wewnątrz sklonowanego katalogu.

```sh
xcx pax clone snake_game
cd snake_game
xcx pax install
xcx pax run
```

## Struktura katalogów

Standardowy projekt PAX ma następujący układ:
- `project.pax`: Konfiguracja.
- `src/`: Kod źródłowy (główne wejście: `main.xcx`).
- `lib/`: Pobrane zależności (zarządzane przez PAX).
- `tests/`: Testy specyficzne dla projektu.
