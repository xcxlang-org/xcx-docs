# xcx-docs
Official documentation repository for the [XCX programming language](https://xcxlang.com), available in multiple languages.

## 📁 Structure
```
xcx-docs/
├── en/        # English          (complete)
├── pl/        # Polish           (complete)
├── fr/        # French           (complete)
├── ru/        # Russian          (complete)
├── zh/        # Chinese (Simp.)  (complete)
├── ja/        # Japanese         (complete)
├── de/        # German           (planned)
├── uk/        # Ukrainian        (planned)
├── ko/        # Korean           (planned)
├── es/        # Spanish          (complete)
├── nl/        # Dutch            (planned)
├── he/        # Hebrew           (planned)
└── ar/        # Arabic           (planned)
```

Each language directory mirrors the same structure:

```text
<lang>/
├── README.md
├── compiler/
│   ├── README.md
│   ├── compiler/
│   ├── diagnostics/
│   ├── frontend/
│   ├── jit/
│   ├── repl/
│   ├── runtime/
│   ├── sema/
│   └── vm/
├── language/
│   ├── collections.md
│   ├── control_flow.md
│   ├── database.md
│   ├── dates.md
│   ├── errors_halt.md
│   ├── functions_fibers.md
│   ├── io_terminal.md
│   ├── json_http.md
│   ├── library_modules.md
│   ├── operators.md
│   ├── string_methods.md
│   ├── syntax.md
│   ├── types.md
│   └── variables.md
└── tooling/
    ├── doc_manual.md
    └── pax_manual.md
```

## 📖 Contents

> ⚠️ **Non-English translations currently reflect older versions, not 4.1.** The process of updating them to 4.1 is underway. Until that is complete, please use the English docs (`en/`) for accurate, up-to-date information.

**Language Reference** covers syntax, types, variables, operators, control flow, functions, fibers, collections, JSON, HTTP, I/O, dates, cryptography, error handling, the standard library, and the PAX package manager.

**Compiler Internals** covers the full compilation pipeline: lexer, Pratt parser, expander, semantic analysis, bytecode compiler, register-based VM, and the Cranelift tracing JIT.

## 🌐 Language Status

| Language | Code | Status |
|---|---|---|
| English | `en` | ✅ complete |
| Polish | `pl` | ✅ complete |
| French | `fr` | ✅ complete |
| Russian | `ru` | ✅ complete |
| Chinese (Simplified) | `zh` | ✅ complete |
| Japanese | `ja` | ✅ complete |
| German | `de` | 🚧 planned |
| Ukrainian | `uk` | 🚧 planned |
| Korean | `ko` | 🚧 planned |
| Spanish | `es` | ✅ complete |
| Dutch | `nl` | 🚧 planned |
| Hebrew | `he` | 🚧 planned |
| Arabic | `ar` | 🚧 planned |

## ⚠️ Translation Notice

Non-English translations were generated with AI assistance and may contain inaccuracies. The English version (`en/`) is always the canonical reference.

Translations are currently being updated to XCX 4.1. Non-English versions should be treated as outdated until this process is complete.

## 🤝 Contributing

When adding or updating a translation, mirror the full directory structure from `en/` exactly. Keep all code blocks, XCX syntax examples, and error codes untranslated. Open a PR with the language code as a label (e.g. `lang:de`).

## 🔗 Links

- Website: [xcxlang.com](https://xcxlang.com)
- GitHub: [github.com/xcxlang-org/xcx](https://github.com/xcxlang-org/xcx)
- Canonical docs: [en/README.md](en/README.md)