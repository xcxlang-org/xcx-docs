# Podręcznik Menedżera Pakietów PAX

PAX to oficjalny menedżer pakietów XCX, zintegrowany bezpośrednio z binarką `xcx`. Zarządza zależnościami, tworzeniem struktury projektów, publikowaniem w rejestrze oraz automatyzacją budowania.

---

## Konfiguracja projektu: `project.pax`

Każdy projekt PAX musi zawierać plik `project.pax` w katalogu głównym. Używa własnego formatu deklaratywnego.

```pax
---
PAX Project Configuration
*---
/
    name        :: "my_project",
    version     :: "1.0.0",
    author      :: "yourname",
    description :: "A short description of the project",
    main        :: "src/main.xcx",
    tags  :: ["tag1", "tag2"],
    files :: ["src/main.xcx", "src/lib.xcx"],
    deps  :: [
        "somelib@1.0.0",
        "author/repo@latest",
        "https://example.com/lib.xcx"
    ]
/
```

### Pola

| Pole          | Wymagane | Opis                                                                         |
|---------------|----------|------------------------------------------------------------------------------|
| `name`        | Tak      | Logiczna nazwa projektu/pakietu.                                             |
| `version`     | Tak      | Ciąg wersji semantycznej (np. `"1.0.0"`).                                    |
| `author`      | Nie      | Nazwa autora, widoczna w rejestrze.                                          |
| `description` | Nie      | Krótki opis projektu.                                                        |
| `main`        | Nie      | Własny punkt wejścia. Domyślnie `src/main.xcx`, jeśli pominięty.            |
| `tags`        | Nie      | Lista tagów ułatwiająca znajdowanie pakietu w rejestrze.                     |
| `files`       | Nie      | Jawna lista plików do dołączenia przy publikowaniu. Jeśli pominięta, PAX automatycznie wyklucza pliki niekodowe. |
| `deps`        | Nie      | Lista zależności. Obsługuje nazwy z wersjami, skróty `author/repo` i bezpośrednie URL. |

### Formaty zależności

```pax
deps :: [
    "mylib@1.0.0",           --- pakiet z rejestru z konkretną wersją
    "mylib@latest",          --- najnowsza wersja z rejestru
    "author/mylib@2.1.0",    --- z prefiksem autora
    "https://example.com/lib.xcx"  --- bezpośredni URL
]
```

---

## Plik blokady: `pax.lock`

PAX automatycznie utrzymuje plik `pax.lock` po każdej instalacji. Rejestruje dokładną rozwiązaną wersję i lokalną ścieżkę każdej zainstalowanej zależności.

```json
{
  "mylib": { "version": "mylib@1.0.0", "file": "lib/mylib/" },
  "otherlib": { "version": "otherlib@latest", "file": "lib/otherlib/" }
}
```

- **Nie edytuj** `pax.lock` ręcznie.
- Commituj go do systemu kontroli wersji dla odtwarzalnych buildów.

---

## Struktura katalogów

Standardowy projekt PAX ma następującą strukturę:

```
my_project/
├── project.pax       # Konfiguracja projektu
├── pax.lock          # Automatycznie generowany plik blokady
├── src/
│   └── main.xcx      # Główny punkt wejścia
├── lib/              # Zainstalowane zależności (zarządzane przez PAX)
└── tests/            # Testy specyficzne dla projektu
```

---

## Dokumentacja poleceń

PAX wywołuje się przez `xcx pax <polecenie>`.

| Polecenie                    | Opis                                                   |
|------------------------------|--------------------------------------------------------|
| `xcx pax new <name>`         | Utwórz nową strukturę projektu.                        |
| `xcx pax install`            | Pobierz wszystkie zależności do `lib/`.                |
| `xcx pax add <dep>`          | Dodaj zależność i od razu ją zainstaluj.               |
| `xcx pax remove <name>`      | Usuń zależność z `project.pax`.                        |
| `xcx pax clone <package>`    | Pobierz opublikowany pakiet jako lokalny projekt.      |
| `xcx pax run [path]`         | Uruchom punkt wejścia projektu.                        |
| `xcx pax search <query>`     | Szukaj dostępnych pakietów w rejestrze.                |
| `xcx pax login <token>`      | Zapisz token uwierzytelniania rejestru.                |
| `xcx pax logout`             | Usuń zapisany token uwierzytelniania.                  |
| `xcx pax whoami`             | Sprawdź aktualne konto w rejestrze.                    |
| `xcx pax publish`            | Opublikuj projekt w rejestrze PAX.                     |

---

## Szczegóły poleceń

### `xcx pax new <name>`

Tworzy nowy projekt ze standardową strukturą katalogów.

```sh
xcx pax new my_project
```

Tworzy:
```
my_project/
├── project.pax
├── src/
│   └── main.xcx
└── README.md
```

---

### `xcx pax install`

Czyta `project.pax`, rozwiązuje wszystkie `deps` i pobiera je do `lib/`. Pomija już istniejące pakiety (po nazwie).

```sh
xcx pax install
```

- Instaluje tylko pliki zadeklarowane w polu `files` danej zależności.
- Jeśli zależność nie ma pola `files`, PAX automatycznie wyklucza `README.md`, `LICENSE`, `tests/`, `docs/`, `.git*` i `project.pax`.
- Aktualizuje `pax.lock` po każdym udanym pobraniu.

---

### `xcx pax add <dep>`

Dodaje zależność do `project.pax` i od razu uruchamia instalację.

```sh
xcx pax add mylib@1.0.0
xcx pax add author/mylib@latest
```

---

### `xcx pax remove <name>`

Usuwa wpis zależności z `project.pax`.

```sh
xcx pax remove mylib
```

> **Uwaga:** Foldery `lib/` i wpisy `pax.lock` nie są automatycznie czyszczone. Uruchom ponownie `xcx pax install` lub usuń je ręcznie.

---

### `xcx pax clone <package>`

Pobiera opublikowany pakiet z rejestru PAX jako kompletny lokalny projekt, gotowy do uruchomienia lub modyfikacji.

```sh
xcx pax clone snake_game
xcx pax clone author/snake_game
```

- Tworzy katalog nazwany po pakiecie w bieżącym katalogu roboczym.
- Kopiuje pełną strukturę projektu, w tym `project.pax`, `src/` i wszystkie zadeklarowane pliki.
- **Nie** instaluje zależności automatycznie.

**Typowy przepływ pracy:**
```sh
xcx pax clone snake_game
cd snake_game
xcx pax install
xcx pax run
```

---

### `xcx pax run [path]`

Uruchamia projekt. Kolejność rozwiązywania punktu wejścia:

1. Ścieżka podana jako argument (np. `xcx pax run other/main.xcx`)
2. Pole `main` w `project.pax`
3. Domyślnie: `src/main.xcx`

```sh
xcx pax run
xcx pax run src/alt.xcx
```

---

### `xcx pax search <query>`

Przeszukuje rejestr PAX w poszukiwaniu pakietów pasujących do zapytania.

```sh
xcx pax search json
```

Format wyjścia:
```
- json_utils (v1.2.0) by alice
- fast_json (v0.9.1) by bob
```

---

### `xcx pax login <token>`

Zapisuje token uwierzytelniania rejestru lokalnie w `.pax_token`.

```sh
xcx pax login your_token_here
```

---

### `xcx pax logout`

Usuwa zapisany token uwierzytelniania.

```sh
xcx pax logout
```

---

### `xcx pax whoami`

Weryfikuje bieżącą sesję rejestru i wyświetla informacje o koncie.

```sh
xcx pax whoami
# Account: alice [developer]
```

---

### `xcx pax publish`

Publikuje bieżący projekt w rejestrze PAX. Wymaga zalogowania.

```sh
xcx pax publish
```

- Odczytuje metadane z `project.pax` (`name`, `version`, `description`).
- Skanuje i przesyła wszystkie pliki projektu (z wyłączeniem `.zip`, `.pax_token`, `.git`).
- Wysyła skompresowane archiwum i manifest plików do rejestru.

> **Uwaga:** Zwiększ `version` w `project.pax` przed każdą publikacją. Rejestr może odrzucić zduplikowane wersje.

---

## Konfiguracja rejestru

Domyślnie PAX łączy się z `pax.xcxlang.com`. Możesz to nadpisać przez plik `.pax_config` w katalogu głównym projektu:

```json
{
  "registry": "https://my-custom-registry.example.com"
}
```

- HTTP jest używane automatycznie dla `localhost` i `127.0.0.1`.
- HTTPS jest używane dla wszystkich pozostałych hostów.
