# Zestaw dokumentacji technicznej XCX

> 📌 **Uwaga**: Ta polska dokumentacja została przetłumaczona przez AI i może zawierać nieścisłości lub błędy tłumaczenia. Aby uzyskać najbardziej precyzyjne informacje, zapoznaj się z [wersją angielską dokumentacji](../en/README.md).

Witamy w oficjalnej dokumentacji technicznej XCX 3.0 i kompilatora XCX.

## 📖 Referencja języka
Kompleksowe przewodniki dotyczące składni i funkcji języka XCX.

- [Podstawy składni](language/syntax.md): Komentarze, identyfikatory i bloki.
- [Zmienne i stałe](language/variables.md): Deklaracje, ponowne przypisanie i zasłanianie zmiennych.
- [Typy danych](language/types.md): Typy proste, typy złożone i rzutowanie.
- [Operatory](language/operators.md): Arytmetyka, łańcuchy znaków, logika, porównania i zbiory.
- [Przepływ sterowania](language/control_flow.md): Instrukcje warunkowe i pętle.
- [Funkcje i włókna](language/functions_fibers.md): Podprogramy, korutyny i delegacja.
- [Kolekcje](language/collections.md): Tablice, zbiory, mapy i tabele (relacyjne).
- [JSON i HTTP](language/json_http.md): Wymiana danych, sieć i bezpieczeństwo.
- [Data i czas](language/dates.md): Tworzenie, pola i formatowanie.
- [I/O i terminal](language/io_terminal.md): Wejście, wyjście, opóźnienia i polecenia systemowe.
- [Metody łańcuchów znaków](language/string_methods.md): Pełna lista narzędzi do obsługi łańcuchów znaków.
- [Obsługa błędów](language/errors_halt.md): Poziomy halt (alert, error, fatal).
- [Biblioteka standardowa](language/library_modules.md): Wbudowane moduły (`crypto`, `store`, `env`).

## ⚙️ Wewnętrzne mechanizmy kompilatora
Szczegółowe omówienie działania kompilatora XCX i maszyny wirtualnej.

- [Przegląd architektury](compiler/architecture.md): Potok kompilacji.
- [Lekser](compiler/lexer.md): Tokenizacja i skanowanie rekurencyjne.
- [Parser](compiler/parser.md): Parsowanie Pratt i generowanie AST.
- [Analiza semantyczna](compiler/semantics.md): Sprawdzanie typów i rozwiązywanie symboli.
- [Maszyna wirtualna](compiler/vm.md): Architektura maszyny stosowej i OpCodes.

## 📦 Narzędzia
Przewodniki dotyczące narzędzi ekosystemu XCX.

- [Podręcznik PAX](pax/pax_manual.md): Zarządzanie pakietami i struktura projektu.

---
*Wersja dokumentacji: 3.0.0 *
