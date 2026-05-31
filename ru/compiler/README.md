# Компилятор XCX — документация v3.1

> Компилятор языка XCX на Rust. Многостадийный конвейер: lexer → parser → семантический анализ → компилятор байткода → виртуальная машина → JIT (Cranelift).

---

## Содержание

1. [Обзор архитектуры](#обзор-архитектуры)
2. [Быстрый старт](#быстрый-старт)
3. [Структура проекта](#структура-проекта)
4. [Конвейер компиляции](#конвейер-компиляции)
5. [Модули](#модули)
   - [Лексер (Scanner)](#лексер-scanner)
   - [Парсер (Pratt)](#парсер-pratt)
   - [Expander](#expander)
   - [Семантический анализ (Sema)](#семантический-анализ-sema)
   - [Компилятор (Backend)](#компилятор-backend)
   - [Виртуальная машина (VM)](#виртуальная-машина-vm)
   - [JIT (Cranelift)](#jit-cranelift)
6. [Ключевые проектные решения](#ключевые-проектные-решения)
7. [Система диагностики](#система-диагностики)
8. [Безопасность](#безопасность)

---

## Обзор архитектуры

```
Source Code (.xcx)
        │
        ▼
  1. Lexer         src/lexer/scanner.rs       → token stream
        │
        ▼
  2. Parser        src/parser/pratt.rs        → raw AST (Program)
        │
        ▼
  3. Expander      src/parser/expander.rs     → expanded AST (include, aliases)
        │
        ▼
  4. Sema          src/sema/checker.rs        → validated, annotated AST
        │
        ▼
  5. Compiler    src/backend/mod.rs         → FunctionChunk + constants + functions
        │
        ▼
  6. VM            src/backend/vm.rs          → bytecode execution (register-based)
        │  hot loops → trace recording
        ▼
  7. JIT           src/backend/jit.rs         → native machine code (Cranelift)
```

---

## Быстрый старт

```bash
# Запуск REPL
xcx

# Запуск файла
xcx program.xcx

# Версия
xcx --version

# Справка
xcx --help
```

**В REPL:**

```
xcx> !help     # справка
xcx> !clear    # очистка экрана
xcx> !exit     # выход
```

---

## Структура проекта

```
src/
├── lexer/
│   ├── scanner.rs        # Сканер байтов (&[u8])
│   └── token.rs          # TokenKind и Span
├── parser/
│   ├── pratt.rs          # Парсер Pratt (токены → AST)
│   ├── expander.rs       # Разрешение include и префиксы псевдонимов
│   └── ast.rs            # Узлы AST (Expr, Stmt, Type, Program)
├── sema/
│   ├── checker.rs        # Проверка типов и разрешение переменных
│   ├── symbol_table.rs   # Иерархическая таблица символов
│   └── interner.rs       # Интернирование строк (str → StringId)
├── backend/
│   ├── mod.rs            # Компилятор байткода (AST → OpCode)
│   ├── vm.rs             # Регистровая VM, NaN-boxing, хуки JIT
│   ├── jit.rs            # Трассовый компилятор Cranelift
│   └── repl.rs           # Интерактивный REPL
└── diagnostic/
    └── report.rs         # Отчёты об ошибках с подсветкой
```

---

## Конвейер компиляции

### 1. Лексер
Преобразует байты исходника в токены. Работа на `&[u8]` — без аллокации `Vec<char>`.

### 2. Парсер
Парсер Pratt строит AST. Один токен заглядывания. Восстановление через `synchronize()`.

### 3. Expander
**После** разбора, **до** семантики. Разрешает `include` и префиксы псевдонимов.

### 4. Семантический анализ
Проверка типов, неопределённые переменные, контекст файберов/циклов. Все ошибки собираются до генерации байткода.

### 5. Компилятор
Два прохода. Проход 1: глобальные и функции. Проход 2: байткод с аннотациями Span.

### 6. VM
Регистровая виртуальная машина. Плоский `Vec<Value>` на кадр, слоты `u8`. Значения 8 байт (NaN-boxing).

### 7. JIT
Автоматическая компиляция горячих циклов в нативный код через Cranelift. Прозрачно для разработчика.

---

## Модули

Подробные описания в отдельных файлах:

- [`lexer.md`](lexer.md) — лексер / сканер
- [`parser.md`](parser.md) — парсер Pratt и AST
- [`expander.md`](expander.md) — Expander
- [`sema.md`](sema.md) — семантический анализ
- [`backend.md`](backend.md) — компилятор и VM
- [`jit.md`](jit.md) — JIT Cranelift
- [`language.md`](language.md) — справочник языка XCX

---

## Ключевые проектные решения

### NaN-boxing значений
Каждое значение — одно `u64` (8 байт). Старшие биты тихого NaN IEEE 754 — теги типа. Нижние 48 бит — полезная нагрузка. Без выделения кучи для скаляров.

### Регистровая VM
Вместо стека операндов — плоский `Vec<Value>` на кадр. Опкоды ссылаются на регистры напрямую.

### Трассовый JIT (Cranelift)
Обратные переходы считаются по IP. После 50 итераций — запись трассы и компиляция. Стражи типов при сбое возвращают в интерпретатор.

### Интернирование строк
Идентификаторы и литералы → `StringId (u32)` через `Interner`.

### Двухпроходная компиляция
`register_globals_recursive` назначает индексы до эмиссии байткода — взаимная рекурсия и вызов до объявления.

---

## Система диагностики

`Reporter` в `src/diagnostic/report.rs`:

```
ERROR: [S101] Undefined variable: foo
   12 | i: x = foo + 1;
              ~~~
```

Каждая ошибка: уровень (ERROR/HALT), место (строка/столбец), подсветка `~~~`.

`TypeError` собираются в `Vec` и выводятся разом до генерации байткода.

---

## Безопасность

### Защита от SSRF в сети
Заблокировано:
- URL `file://`
- `169.254.x.x` (link-local / метаданные AWS)
- Частные диапазоны: `10.x`, `192.168.x`, `172.16–31.x` (кроме localhost)

Для `HttpCall` и `HttpRequest`.

### Лимит размера HTTP
В `HttpServe` после `into_string()`. Свыше **10 МБ** — JSON-ошибка 413.

### Заголовки CORS
Ответы `HttpServe` по умолчанию включают:
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

Preflight `OPTIONS` — ответ `204` без вызова обработчика.

### Безопасные пути файлов
`store.*` блокирует `..`, абсолютные пути и буквы дисков Windows.
