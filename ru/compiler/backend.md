# XCX Backend — компилятор и VM

> **Файлы:** `src/backend/mod.rs`, `src/backend/vm.rs`

---

## Содержание

1. [Компилятор байткода](#компилятор-байткода)
2. [Представление значений — NaN-boxing](#представление-значений--nan-boxing)
3. [Набор инструкций (OpCodes)](#набор-инструкций-opcodes)
4. [Архитектура VM](#архитектура-vm)
5. [Модель выполнения файберов](#модель-выполнения-файберов)
6. [HTTP-сервер](#http-сервер)
7. [Управление памятью](#управление-памятью)
8. [Оптимизации циклов](#оптимизации-циклов)

---

## Компилятор байткода

**Файл:** `src/backend/mod.rs`

### Распределение регистров

`FunctionCompiler` отслеживает следующий свободный регистр через `next_local: usize`:

```rust
pub fn push_reg(&mut self) -> u8 {
    let r = self.next_local as u8;
    self.next_local += 1;
    if self.next_local > self.max_locals_used {
        self.max_locals_used = self.next_local;
    }
    r
}

pub fn pop_reg(&mut self) {
    self.next_local -= 1;
}
```

Именованные локальные переменные привязываются к слоту через `define_local(id, slot)` и хранятся в `scopes: Vec<HashMap<StringId, usize>>`. Временные переменные используют `push_reg()`/`pop_reg()` — переиспользуются, когда результат выражения больше не нужен.

`max_locals_used` записывается, чтобы `FunctionChunk::max_locals` мог заранее выделить вектор locals нужного размера при вызове функции.

### Дедупликация констант

`CompileContext::add_constant` дедуплицирует строковые константы через `string_constants: HashMap<String, usize>`. Повторяющиеся строки (например, `"insert"` как имя метода) используют один слот в таблице констант.

### Двухпроходная компиляция

**Проход 1** — `register_globals_recursive`:
- Назначает индекс слота каждой глобальной переменной и экземпляру объявления файбера
- Назначает индекс функции каждой функции/файберу
- Предварительно выделяет пустые слоты `FunctionChunk` в `functions: Vec<FunctionChunk>`

**Проход 2** — `compile_stmt` / `compile_expr`:
- Эмитирует байткод в паре со Span через `emit(op, span)`
- Инструкции верхнего уровня в `main` используют `GetVar`/`SetVar` (глобальные); вложенные — регистры

### FunctionChunk

```rust
pub struct FunctionChunk {
    pub bytecode:   Arc<Vec<OpCode>>,
    pub spans:      Arc<Vec<Span>>,    // spans[i] corresponds to bytecode[i]
    pub is_fiber:   bool,
    pub max_locals: usize,
}
```

Bytecode и spans обёрнуты в `Arc` для совместного использования HTTP-воркерами без копирования.

---

## Представление значений — NaN-boxing

Каждое значение — одно `Value(u64)` — 64-битное слово. XCX использует **NaN-boxing**: битовый шаблон тихого NaN IEEE 754 служит префиксом тега типа.

```
Bit layout: [63..52: exponent/QNAN] [51..48: type tag] [47..0: payload]

Float : stored directly as f64 bits — DOES NOT have the QNAN_BASE prefix set
Int   : QNAN_BASE | TAG_INT  | (i48 value & 0x0000_FFFF_FFFF_FFFF)
Bool  : QNAN_BASE | TAG_BOOL | (0 or 1)
Date  : QNAN_BASE | TAG_DATE | (i48 ms timestamp)
Ptr   : QNAN_BASE | TAG_XXX  | (pointer & 0x0000_FFFF_FFFF_FFFF)
```

### Константы тегов

| Константа | Значение | Тип |
|---|---|---|
| `QNAN_BASE` | `0x7FF0_0000_0000_0000` | базовый маркер NaN |
| `TAG_INT` | `0x0001_0000_0000_0000` | 48-битное целое со знаком |
| `TAG_BOOL` | `0x0002_0000_0000_0000` | логическое (payload 0/1) |
| `TAG_DATE` | `0x0003_0000_0000_0000` | 48-битная метка времени (мс) |
| `TAG_STR` | `0x0004_0000_0000_0000` | указатель `Arc<Vec<u8>>` |
| `TAG_ARR` | `0x0005_0000_0000_0000` | указатель `Arc<RwLock<Vec<Value>>>` |
| `TAG_SET` | `0x0006_0000_0000_0000` | указатель `Arc<RwLock<SetData>>` |
| `TAG_MAP` | `0x0007_0000_0000_0000` | указатель `Arc<RwLock<Vec<(Value,Value)>>>` |
| `TAG_TBL` | `0x0008_0000_0000_0000` | указатель `Arc<RwLock<TableData>>` |
| `TAG_FUNC` | `0x0009_0000_0000_0000` | индекс функции (u32) |
| `TAG_ROW` | `0x000A_0000_0000_0000` | указатель `Arc<RowRef>` |
| `TAG_JSON` | `0x000B_0000_0000_0000` | указатель `Arc<RwLock<serde_json::Value>>` |
| `TAG_FIB` | `0x000C_0000_0000_0000` | указатель `Arc<RwLock<FiberState>>` |
| `TAG_DB` | `0x000D_0000_0000_0000` | указатель `Arc<DatabaseData>` |

Полезная нагрузка указателей использует только младшие 48 бит — на x86-64 и AArch64 указатели пользовательского пространства укладываются в 48 бит.

### Подсчёт ссылок

Значения с тегом указателя несут счётчики ссылок `Arc`. VM управляет ими вручную через `inc_ref()` / `dec_ref()` при присваивании, возврате и изменении коллекций — освобождая объекты в куче без сборщика мусора.

---

## Набор инструкций (OpCodes)

Все опкоды регистровые: ссылаются на слоты `u8`, а не на стек операндов.

### Перемещение регистров / переменных

| OpCode | Описание |
|---|---|
| `LoadConst { dst, idx }` | Загрузить `constants[idx]` в регистр `dst` |
| `Move { dst, src }` | Скопировать `src` в `dst` |
| `GetVar { dst, idx }` | Загрузить `globals[idx]` в `dst` (read-lock) |
| `SetVar { idx, src }` | Записать `src` в `globals[idx]` (write-lock) |

### Арифметика

Все операции 3-регистровые: `dst = src1 OP src2`. Диспетчеризация типов в runtime выбирает пути для целых, float, конкатенации строк, дат или операций над множествами.

`Add`, `Sub`, `Mul`, `Div`, `Mod`, `Pow`, `IntConcat` (`++`)

### Сравнения (результат Bool)

`Equal`, `NotEqual`, `Greater`, `Less`, `GreaterEqual`, `LessEqual`

### Логика

`And { dst, src1, src2 }`, `Or { dst, src1, src2 }`, `Not { dst, src }`, `Has { dst, src1, src2 }`

### Управление потоком

| OpCode | Описание |
|---|---|
| `Jump { target }` | Безусловный переход; увеличивает `hot_counts[target]` при обратных переходах |
| `JumpIfFalse { src, target }` | Переход, если `src` — `Bool(false)` |
| `JumpIfTrue { src, target }` | Переход, если `src` — `Bool(true)` |
| `Call { dst, func_idx, base, arg_count }` | Вызов функции; результат → `dst` |
| `Return { src }` | Вернуть значение из `src` |
| `ReturnVoid` | Вернуться без значения |
| `Halt` | Остановить выполнение |

### Коллекции

| OpCode | Описание |
|---|---|
| `ArrayInit { dst, base, count }` | Собрать `count` регистров с `base` → массив в `dst` |
| `SetInit { dst, base, count }` | Собрать `count` регистров → множество в `dst` |
| `SetRange { dst, start, end, step, has_step }` | Множество-диапазон из регистров |
| `MapInit { dst, base, count }` | Собрать пары ключ–значение → карта в `dst` |
| `TableInit { dst, skeleton_idx, base, row_count }` | Таблица из схемы столбцов и строк |

### Операции над множествами

`SetUnion`, `SetIntersection`, `SetDifference`, `SetSymDifference` — 3-регистровые, оба операнда `TAG_SET`.

### Диспетчеризация методов

| OpCode | Описание |
|---|---|
| `MethodCall { dst, kind, base, arg_count }` | Встроенный метод через `MethodKind` — без поиска строки в runtime |
| `MethodCallCustom { dst, method_name_idx, base, arg_count }` | Динамический метод (JSON, alias) из таблицы констант |
| `MethodCallNamed { dst, kind, base, arg_count, names_idx }` | Вызов с именованными аргументами |

`base` — регистр получателя; аргументы в `locals[base+1..base+1+arg_count]`. `MethodKind` — enum ~50 встроенных методов (`Push`, `Pop`, `Get`, …).

### Операции с файберами

| OpCode | Описание |
|---|---|
| `FiberCreate { dst, func_idx, base, arg_count }` | Создать `FiberState`, заполнить locals, сохранить в `dst` |
| `Yield { src }` | Приостановить файбер, вернуть значение вызывающему |
| `YieldVoid` | Приостановить пустой файбер |

### Ввод-вывод и система

| OpCode | Описание |
|---|---|
| `Print { src }` | Вывод `locals[src]` в stdout |
| `Input { dst, ty }` | Строка из stdin → `dst` с приведением типа |
| `Wait { src }` | Пауза `src` миллисекунд |
| `HaltAlert { src }` | Предупреждение, выполнение продолжается |
| `HaltError { src }` | Ошибка + span, остановка кадра |
| `HaltFatal { src }` | Fatal + span, остановка кадра |
| `TerminalExit` | `std::process::exit(0)` |
| `TerminalClear` | Очистка терминала (ANSI) |
| `TerminalRaw / TerminalNormal` | Сырой / обычный режим терминала |
| `TerminalCursor { on }` | Показать/скрыть курсор |
| `TerminalMove { x_src, y_src }` | Переместить курсор |
| `TerminalWrite { src }` | Вывод без перевода строки |
| `InputKey { dst }` | Клавиша (неблокирующе) |
| `InputKeyWait { dst }` | Клавиша (с ожиданием) |
| `InputReady { dst }` | Есть ли ввод |
| `EnvGet { dst, src }` | Переменная окружения |
| `EnvArgs { dst }` | Аргументы CLI |

### HTTP

| OpCode | Описание |
|---|---|
| `HttpCall { dst, method_idx, url_src, body_src }` | Простой HTTP через `ureq`, JSON → `dst` |
| `HttpRequest { dst, arg_src }` | Полный HTTP из карты конфигурации → `dst` |
| `HttpRespond { status_src, body_src, headers_src }` | Ответ HTTP из обработчика файбера |
| `HttpServe { func_idx, port_src, host_src, workers_src, routes_src }` | Сервер `tiny_http`, рабочие потоки |

### Хранилище (store)

`StoreWrite`, `StoreRead`, `StoreAppend`, `StoreExists`, `StoreDelete`, `StoreList`, `StoreIsDir`, `StoreSize`, `StoreMkdir`, `StoreGlob`, `StoreZip`, `StoreUnzip`

### JSON

`JsonParse { dst, src }`, `JsonBind { idx, json_src, path_src }`, `JsonBindLocal { dst, json_src, path_src }`, `JsonInject { table_idx, json_src, mapping_src }`, `JsonInjectLocal { table_reg, json_src, mapping_src }`

### Приведение типов

`CastInt { dst, src }`, `CastFloat { dst, src }`, `CastString { dst, src }`, `CastBool { dst, src }`

### Криптография и даты

`CryptoHash`, `CryptoVerify`, `CryptoToken`, `DateNow`

### База данных

`DatabaseInit { dst, engine_src, path_src, tables_base_reg, table_count }`

---

## Архитектура VM

**Файл:** `src/backend/vm.rs`

### Состояние VM

```rust
pub struct VM {
    pub globals:     Arc<RwLock<Vec<Value>>>,
    pub error_count: AtomicUsize,
    pub traces:      Arc<RwLock<HashMap<usize, Arc<Trace>>>>,
    pub jit:         Mutex<JIT>,
}
```

`VM` обёрнут в `Arc<VM>` и разделяется HTTP-воркерами. Каждый воркер создаёт свой `Executor` с собственными locals.

### Состояние Executor

```rust
struct Executor {
    vm:               Arc<VM>,
    ctx:              SharedContext,
    current_spans:    Option<Arc<Vec<Span>>>,
    fiber_yielded:    bool,
    hot_counts:       Vec<usize>,         // IP backward jump counter
    recording_trace:  Option<Trace>,      // trace being recorded
    is_recording:     bool,
    trace_cache:      Vec<Option<Arc<Trace>>>, // compiled traces indexed by start IP
    http_req:         Option<Arc<Mutex<Option<tiny_http::Request>>>>,
    http_req_val:     Option<Value>,
    terminal_raw_enabled: bool,
}
```

### SharedContext

```rust
pub struct SharedContext {
    pub constants: Arc<Vec<Value>>,
    pub functions: Arc<Vec<FunctionChunk>>,
}
```

`SharedContext` дёшево клонируется (два bump счётчика `Arc`) и передаётся каждому воркеру без глубокого копирования.

### Поток выполнения

```
VM::run(main_chunk, ctx)                        [Arc<VM>]
  └─ Executor::run_frame_owned(main_chunk)
       └─ execute_bytecode_inner(bytecode, &mut ip, &mut locals, ...)
            │
            ├─ [JIT Fast Path] if trace_cache[ip].is_some():
            │     execute_trace(trace, ip, locals, globals)
            │     → returns next IP or None
            │
            └─ [Interpreter Path] fetch opcode, execute
                 ├─ Continue      → proceed normally
                 ├─ Jump(t)       → ip = t; increment hot_counts if backward
                 ├─ Return(val)   → exit frame, return val
                 ├─ Yield(val)    → suspend (fiber), return val to caller
                 └─ Halt          → stop, increment error_count
```

---

## Модель выполнения файберов

Файберы — **кооперативные корутины**, не потоки ОС.

```rust
pub struct FiberState {
    pub func_id:       usize,
    pub ip:            usize,
    pub locals:        Vec<Value>,    // moved during resume, returned after
    pub is_done:       bool,
    pub yielded_value: Option<Value>, // cache for IsDone + Next pattern
}
```

### Последовательность возобновления (`resume_fiber`)

1. Прочитать `func_id`, `ip` и **перенести** `locals` из `FiberState` через `std::mem::take` — без клонирования
2. Запустить `execute_bytecode` с `fiber.ip` и перенесёнными locals
3. При `Yield`: `fiber_yielded = true`, locals обратно в `FiberState`, обновить `fiber.ip`, вернуть значение
4. При `Return` / конце байткода: `fiber.is_done = true`, вернуть итог

Возобновление/приостановка не аллоцирует в куче кроме начального `Vec` — только перемещения.

### Шаблон IsDone / Next

`IsDone` проверяет `FiberState::is_done` и кэш `yielded_value`. `Next` берёт кэш или вызывает `resume_fiber`. Цикл `for x in fiber` не продвигает файбер дважды.

### Цикл for по файберу (`ForIterType::Fiber`)

Компилятор эмитирует:
1. `MethodCall(IsDone)` → `JumpIfTrue` на выход
2. `MethodCall(Next)` → переменная цикла
3. Тело цикла
4. `Jump` к шагу 1
5. При `break`: `MethodCall(Close, base = fiber_reg)` перед переходом

---

## HTTP-сервер

`HttpServe` запускает `tiny_http` и создаёт N потоков ОС:

```rust
for _ in 0..workers {
    let server  = server.clone();    // Arc<tiny_http::Server>
    let vm      = vm_arc.clone();    // Arc<VM>
    let ctx     = self.ctx.clone();  // SharedContext (two Arc clones)
    let routes  = routes.clone();    // Arc<Vec<(String, usize)>>
    std::thread::spawn(move || { /* recv → match route → run handler fiber */ });
}
```

Каждый воркер запускает свой `Executor` с locals. Глобальные — через `Arc<RwLock<Vec<Value>>>`.

### Обработка запросов

Для каждого запроса воркер:
1. Сопоставляет `"METHOD /path"` с таблицей маршрутов (без учёта регистра)
2. Строит JSON `{ method, url, body, ip, headers }` как `Value::Json`
3. Хранит `tiny_http::Request` в `Arc<Mutex<...>>` и передаёт в `Executor`
4. Синхронно запускает обработчик-файбер
5. При `net.respond(...)` — `HttpRespond` отправляет ответ
6. Без `net.respond` — ответ `500`

### Корректное завершение

`SHUTDOWN` — `pub static AtomicBool` в `vm.rs`. Ctrl+C в `main.rs` устанавливает `true`. Воркеры опрашивают каждые `recv_timeout(100ms)`. Главный поток — `sleep(500ms)` и опрос `SHUTDOWN`. Затем чистый выход процесса.

---

## Управление памятью

- **Без GC**. Подсчёт ссылок через `Arc` и ручной `inc_ref`/`dec_ref` для указательных NaN-boxed значений.
- **Скаляры** (Int, Float, Bool, Date, индекс функции): целиком в `u64` — без кучи.
- **Коллекции**: `Arc<RwLock<T>>`. Клон `Value` только увеличивает счётчик `Arc`.
- **Мутации**: write lock на `.insert()`, `.update()`, `.delete()`.
- **Чтение**: read lock на `.size()`, `.get()`, `.contains()` — много читателей допустимо.

---

## Оптимизации циклов

Эти опкоды сливают типичные счётчики цикла в одну инструкцию, снижая накладные расходы и улучшая JIT:

| OpCode | Описание |
|---|---|
| `IncLocal { reg }` | Инкремент целого в регистре `reg` |
| `IncVar { idx }` | Инкремент глобальной по `idx` |
| `LoopNext { reg, limit_reg, target }` | Инкремент `reg`, переход к `target`, если `reg <= limit_reg` |
| `IncLocalLoopNext { ... }` | Отдельный счётчик + переменная цикла + условный переход |
| `IncVarLoopNext { ... }` | Как выше, глобальный счётчик |
| `ArrayLoopNext { ... }` | Итерация по индексу массива |

### Преобразование оптимизации

Перед концом шага цикла компилятор проверяет последний опкод. `IncVar` / `IncLocal` заменяются на `IncVarLoopNext` / `IncLocalLoopNext`:

```rust
match self.bytecode[len - 1] {
    OpCode::IncVar { idx } => {
        self.bytecode.pop();
        self.emit(OpCode::IncVarLoopNext { g_idx: idx, reg: loop_var_reg, ... });
    }
    OpCode::IncLocal { reg } => {
        self.bytecode.pop();
        self.emit(OpCode::IncLocalLoopNext { inc_reg: reg, ... });
    }
}
```
