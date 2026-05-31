# Parser XCX — v3.0

Parser XCX przekształca strumień tokenów w wysokopoziomowe abstrakcyjne drzewo składni (AST).

## Architektura: parsowanie Pratt

XCX używa **parsera Pratt** (Top-Down Operator Precedence).

- **Plik**: `src/parser/pratt.rs`
- **Lookahead**: Jeden token (`current` + `peek`), przesuwany ręcznie przez `advance()`.
- **Odzyskiwanie po błędach**: Przy błędzie składni `synchronize()` pomija tokeny do następnego średnika lub znanego słowa kluczowego rozpoczynającego instrukcję (`func`, `fiber`, `if`, `for`, `const`, `return`, `>!` itd.).

Struktura `Parser` pożycza łańcuch źródłowy na czas życia `'a`, a `Scanner<'a>` parametryzowany jest tym samym czasem życia, odzwierciedlając skaner oparty na wycinku bajtów.

### Poziomy precedencji (od najniższego)

| Poziom | Operatory |
|---|---|
| `Lowest` | — |
| `Lambda` | `->` |
| `Assignment` | `=` |
| `LogicalOr` | `OR`, `\|\|` |
| `LogicalAnd` | `AND`, `&&` |
| `Equals` | `==`, `!=` |
| `LessGreater` | `>`, `<`, `>=`, `<=`, `HAS` |
| `Sum` | `+`, `-`, `++` |
| `SetOp` | `UNION`, `INTERSECTION`, `DIFFERENCE`, `SYMMETRIC_DIFFERENCE` |
| `Product` | `*`, `/`, `%` |
| `Power` | `^` |
| `Prefix` | `-x` |
| `Call` | `.`, `[` |

## Dyspozycja instrukcji

`parse_statement_internal()` dyspozytuje na podstawie bieżącego tokenu:

- **Słowa kluczowe typów** (`i`, `f`, `s`, `b`, `array`, `set`, `map`, `date`, `table`, `json`) → `parse_var_decl()`, lub `parse_assignment()` jeśli następuje `=`
- **`const`** → `parse_var_decl()` z `is_const = true`
- **`var`** (identyfikator) → deklaracja zmiennej z inferencją typu
- **`>!`** → `parse_print_stmt()`
- **`>?`** → `parse_input_stmt()`
- **`halt`** → `parse_halt_stmt()`
- **`if`** → `parse_if_statement()`
- **`while`** → `parse_while_statement()`
- **`for`** → `parse_for_statement()`
- **`break`** / **`continue`** → `parse_break_statement()` / `parse_continue_statement()`
- **`func`** → `parse_func_def()`
- **`fiber`** → `parse_fiber_statement()` (def lub decl w zależności od peek)
- **`return`** → `parse_return_stmt()`
- **`yield`** → `parse_yield_stmt()` (obsługuje `yield expr`, `yield from expr` i `yield;`)
- **`@wait`** → `parse_wait_stmt()`
- **`serve`** → `parse_serve_stmt()`
- **`net`** → `parse_net_stmt()`
- **`include`** → `parse_include_stmt()`
- **Identyfikator + `=`** → `parse_assignment()`
- **Identyfikator + `(`** → `parse_func_call_stmt()`
- **Cokolwiek innego** → `parse_expr_stmt()`

## Style definicji funkcji

XCX obsługuje dwa składniowo różne style definiowania funkcji:

**Styl nawiasowy** (jak w C):
```xcx
func name(i: x, s: y -> i) {
    return x + 1;
}
```

**Styl XCX** (blok słów kluczowych):
```xcx
func:i: name(i: x, s: y) do;
    return x + 1;
end;
```

Oba produkują identyczne węzły AST `StmtKind::FunctionDef`. Typ zwracany w stylu nawiasowym deklarowany jest przez `-> type` w liście parametrów lub po `)`.

## Instrukcje fiber

`parse_fiber_statement()` patrzy na `peek`:
- `peek == Colon` → `parse_fiber_decl()` (instancjonowanie: `fiber:T: varname = fiberDef(args);`)
- w przeciwnym razie → `parse_fiber_def()` (definicja: `fiber name(params) { body }`)

`parse_fiber_decl()` obsługuje też przypadek, gdy po parsowaniu typu i nazwy bieżącym tokenem jest `(` — wtedy przechodzi do `finish_fiber_def()` (definicja z wiodącą adnotacją typu `fiber:`).

## Kluczowe konstrukcje parsowane

- **Deklaracje zmiennych**: `i: name = expr;`, `const s: NAME = expr;`, `var name = expr;`
- **Przepływ sterowania**: `if (cond) then; ... elseif (cond) then; ... else; ... end;`
- **Pętla while**: `while (cond) do; ... end;`
- **Pętla for**: `for x in expr do; ... end;` i `for x in start to end @step n do; ... end;`
- **Funkcje**: `func` (dwa style, patrz wyżej)
- **Fibery**: `fiber name(params) { body }` i `fiber:T: varname = fiberName(args);`
- **Yield**: `yield expr;`, `yield from expr;`, `yield;`
- **HTTP**: `serve: name { port=..., routes=... };`, `net.get(url)`, `net.request { ... } as resp;`, `net.respond(status, body);`
- **Kolekcje**: tablica `[a, b, c]`, zbiór `set:N { 1,,10 }`, mapa `[k :: v, ...]`, tabela `table { columns=[...] rows=[...] }`
- **Bloki raw**: `<<<...>>>` dla inline JSON/łańcuchów
- **Include**: `include "path";` lub `include "path" as alias;`
- **I/O**: `>! expr;` (print), `>? varname;` (input)
- **Halt**: `halt.alert >! msg;`, `halt.error >! msg;`, `halt.fatal >! msg;`
- **Wait**: `@wait(ms);` lub `@wait ms;`
- **Literały dat**: `date("2024-01-01")` lub `date("01/01/2024", "DD/MM/YYYY")`

## Parsowanie wyrażeń

`parse_expression(precedence)` wywołuje `parse_prefix()` dla lewej strony, potem pętlą `parse_infix(left)`, dopóki precedencja tokenu peek przekracza bieżące minimum.

Kluczowe parsery prefiksowe:
- **Identyfikatory**: Jeśli następuje `(`, parsowane jako `FunctionCall`; w przeciwnym razie `Identifier`.
- **Literały**: `IntLiteral`, `FloatLiteral`, `StringLiteral`, `True`, `False`
- **Unary minus**: Parsowane jako `Binary { left: IntLiteral(0), op: Minus, right }` (brak osobnego `Unary::Neg`)
- **`not` / `!`**: `Unary { op: Not/Bang, right }`
- **Grupy `(`...`)`**: Pojedyncze wyrażenie → rozpakowane; wiele rozdzielonych przecinkami → `Tuple`
- **`[`...`]`**: Jeśli pierwszy element po `::`, parsowane jako `MapLiteral`; w przeciwnym razie `ArrayLiteral`
- **`{`...`}`**: Parsowane jako `ArrayOrSetLiteral` (typ rozstrzygany w semantyce lub kompilacji)
- **`set:N { }` itd.**: Jawny `SetLiteral` ze znanym `SetType`
- **`map { schema=[...] data=[...] }`**: Jawny `MapLiteral`
- **`table { columns=[...] rows=[...] }`**: `TableLiteral`
- **`random.choice from expr`**: `RandomChoice`
- **`date(...)`**: `DateLiteral`
- **`net.get/post/put/delete/patch(...)` itd.**: `NetCall` lub `NetRespond`
- **`<<<...>>>`**: `RawBlock`
- **`.terminal!cmd`**: `TerminalCommand`

Kluczowe parsery infiksowe:
- **`.`**: `parse_dot_infix` — `MethodCall` jeśli następuje `(`, w przeciwnym razie `MemberAccess`; obsługuje też dostęp `.[key]`
- **`[`**: `parse_index_infix` → `Index`
- **`->`**: `parse_lambda_infix` → `Lambda`
- **Wszystkie operatory binarne**: `Binary { left, op, right }`

## Post-processing `parse_expr_stmt()`

Po sparsowaniu pełnej instrukcji wyrażenia `parse_expr_stmt()` sprawdza, czy wynik to `MethodCall`:
- Nazwa metody `bind` z 2 argumentami i drugim `Identifier` → przepisanie na `StmtKind::JsonBind`
- Nazwa metody `inject` z 2 argumentami → przepisanie na `StmtKind::JsonInject`

Umożliwia składnię `json.bind("path", target);` i `json.inject(mapping, table);` na poziomie instrukcji.

## Expander (`src/parser/expander.rs`)

Expander uruchamiany jest **po** parsowaniu, **przed** analizą semantyczną. To osobny przebieg przepisywania drzewa.

### Odpowiedzialności

**Rozwiązywanie include**: `include "file.xcx";` zastępowane jest wstawionym AST tego pliku. Zależności cykliczne wykrywane przez `visiting_files: HashSet<PathBuf>`. Pliki deduplikowane przez `included_files: HashSet<PathBuf>` (każdy plik tylko raz, chyba że z aliasem).

**Prefiksowanie aliasów**: `include "math.xcx" as math;` powoduje, że wszystkie nazwy najwyższego poziomu z pliku dostają prefiks `math.name`. Miejsca wywołań (`math.sin(x)`) przepisywane z `MethodCall` na `FunctionCall { name: "math.sin" }` przez `expand_expr_inplace`. Funkcje `prefix_program()` / `prefix_stmt_impl()` / `prefix_expr_impl()` przechodzą całe pod-AST, zmieniając odwołania do symboli najwyższego poziomu.

**Prefiksowanie nazw fiber**: Odwołania `FiberDecl::fiber_name` też są prefiksowane, aby instancjonowanie po zmianie nazw rozwiązywało się poprawnie.

**Prefiksowanie YieldFrom**: Wyrażenia `StmtKind::YieldFrom` są przechodzone, aby wywołania konstruktorów fiber w `yield from` też były przemianowane.

**Chronione nazwy** (nigdy nie prefiksowane): `json`, `date`, `store`, `halt`, `terminal`, `net`, `env`, `crypto`, `EMPTY`, `math`, `random`, `i`, `f`, `s`, `b`, `from`, `main`.

**Kolejność wyszukiwania ścieżek include**:
1. Względem katalogu bieżącego pliku
2. W katalogu `lib/` (względem CWD, potem w górę od ścieżki pliku wykonywalnego)

## Definicje AST (`src/parser/ast.rs`)

### `Expr` — węzły wyrażeń

| Wariant | Opis |
|---|---|
| `IntLiteral(i64)` | Stała całkowita |
| `FloatLiteral(f64)` | Stała zmiennoprzecinkowa |
| `StringLiteral(StringId)` | Internowany łańcuch |
| `BoolLiteral(bool)` | `true` / `false` |
| `Identifier(StringId)` | Nazwa zmiennej lub funkcji |
| `Binary { left, op, right }` | Operacja binarna |
| `Unary { op, right }` | Operacja jednoargumentowa (`not`, `!`) |
| `FunctionCall { name, args }` | Wywołanie funkcji po internowanej nazwie |
| `MethodCall { receiver, method, args }` | Wywołanie przez kropkę na wartości |
| `MemberAccess { receiver, member }` | Dostęp przez kropkę bez wywołania |
| `Index { receiver, index }` | Indeks w nawiasach `a[i]` |
| `Lambda { params, return_type, body }` | Lambda strzałkowa `x -> expr` |
| `ArrayLiteral { elements }` | Jawne `[a, b, c]` |
| `ArrayOrSetLiteral { elements }` | Niejednoznaczne `{a, b, c}` — rozstrzygane później |
| `SetLiteral { set_type, elements, range }` | Typowany zbiór z opcjonalnym zakresem |
| `MapLiteral { key_type, value_type, elements }` | Literał mapy |
| `TableLiteral { columns, rows }` | Literał tabeli |
| `DateLiteral { date_string, format }` | `date("2024-01-01")` |
| `Tuple(Vec<Expr>)` | Lista rozdzielona przecinkami w nawiasach |
| `NetCall { method, url, body }` | Wyrażenie wywołania HTTP |
| `NetRespond { status, body, headers }` | Wyrażenie odpowiedzi HTTP |
| `RawBlock(StringId)` | Surowa treść `<<<...>>>` |
| `TerminalCommand(cmd, arg)` | `.terminal !cmd` |
| `RandomChoice { set }` | `random.choice from set` |

### `Stmt` — węzły instrukcji

Kluczowe warianty: `VarDecl`, `Assign`, `Print`, `Input`, `If`, `While`, `For`, `Break`, `Continue`, `FunctionDef`, `FiberDef`, `FiberDecl`, `Return`, `Yield`, `YieldFrom`, `YieldVoid`, `Include`, `Serve`, `NetRequestStmt`, `JsonBind`, `JsonInject`, `Halt`, `Wait`, `ExprStmt`, `FunctionCallStmt`.

### `Type` — system typów

`Int`, `Float`, `String`, `Bool`, `Date`, `Json`, `Array(Box<Type>)`, `Set(SetType)`, `Map(Box<Type>, Box<Type>)`, `Table(Vec<ColumnDef>)`, `Fiber(Option<Box<Type>>)`, `Builtin(StringId)`, `Unknown`.

Warianty `SetType`: `N` (Natural), `Z` (Integer), `Q` (Rational/Float), `S` (String), `C` (Char/String), `B` (Boolean).

### `ForIterType`

`Range` (numeryczne `start to end`), `Array`, `Set`, `Fiber` — ustawiane przez sprawdzanie typów i używane przez kompilator do emisji właściwego wzorca pętli.

### `ColumnDef`

```rust
pub struct ColumnDef {
    pub name:    StringId,
    pub ty:      Type,
    pub is_auto: bool,    // @auto columns are auto-incremented on insert
}
```

## Interner łańcuchów

Wszystkie wartości łańcuchowe (identyfikatory, literały, nazwy metod) internowane są przez `Interner` do `StringId (u32)`. Interner tworzony jest w parserze i przekazywany przez wszystkie kolejne fazy. Checker, kompilator i VM używają numerycznych ID do porównań nazw zamiast porównań `String`.
