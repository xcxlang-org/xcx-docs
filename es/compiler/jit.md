> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/compiler/jit.md).

# Compilador JIT de Rastreo de XCX (Cranelift) — Documentación

> **Archivo:** `src/backend/jit.rs`  
> XCX 3.0 incluye un **JIT de modo dual** que compila automáticamente bucles calientes y funciones frecuentemente llamadas en código máquina nativo usando el framework [Cranelift](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift).

---

## Tabla de Contenidos

1. [Descripción General](#descripción-general)
2. [Dos Modos JIT](#dos-modos-jit)
3. [JIT de Rastreo — Detección de Bucles Calientes](#jit-de-rastreo--detección-de-bucles-calientes)
4. [Grabación de Rastreo](#grabación-de-rastreo)
5. [Compilación de Rastreo](#compilación-de-rastreo)
6. [Ejecución de Rastreo](#ejecución-de-rastreo)
7. [JIT de Método — Compilación de Función](#jit-de-método--compilación-de-función)
8. [Variantes de TraceOp](#variantes-de-traceop)
9. [Funciones C Exportadas](#funciones-c-exportadas)
10. [Configuración de JIT](#configuración-de-jit)

---

## Descripción General

El JIT es **completamente transparente** para el desarrollador — se activa automáticamente y revierte al intérprete al encontrar guardias de tipo u operaciones no admitidas.

Hay dos rutas de compilación independientes:

```
[JIT de Rastreo] — para bucles calientes
  Intérprete ejecuta bucle
      ↓ (50 iteraciones)
  Grabación de rastreo comienza
      ↓ (bucle retorna al IP inicial)
  Rastreo compilado por Cranelift → función C nativa (firma de 3 argumentos)
      ↓ (siguiente iteración)
  Código nativo ejecutado directamente (omisión de intérprete)
      ↓ (guardia fallida)
  Reversión al intérprete en el IP correcto

[JIT de Método] — para funciones frecuentemente llamadas sin bucles
  Función es llamada
      ↓ (10 llamadas)
  Bytecode completo de función compilado por Cranelift → función C nativa (firma de 5 argumentos)
      ↓ (siguiente llamada)
  Código nativo llamado directamente (omisión de intérprete + maquinaria de rastreo)
```

---

## Dos Modos JIT

### JITFunction (Rastreo)

```rust
pub type JITFunction = unsafe extern "C" fn(
    *mut VMValue,   // puntero_locales
    *mut VMValue,   // puntero_globales
    *const VMValue, // puntero_constantes
) -> i32            // siguiente_ip (0 = continuar, >0 = IP de salida lateral, <0 = detener)
```

Usado para rastreos de bucles calientes. El valor de retorno es el siguiente puntero de instrucción.

### MethodJitFunction (Método)

```rust
pub type MethodJitFunction = unsafe extern "C" fn(
    *mut VMValue,   // puntero_locales
    *mut VMValue,   // puntero_globales
    *const VMValue, // puntero_constantes
    *mut VM,        // puntero_vm
    *mut Executor,  // puntero_executor
) -> u64            // valor de retorno como bits de Value en caja NaN
```

Usado para funciones compiladas completas. Retorna el valor de retorno de la función directamente como bits de NaN-boxing. Necesita acceso a la VM y Executor para llamadas recursivas y despacho de métodos.

---

## JIT de Rastreo — Detección de Bucles Calientes

En cada `Jump { target }` donde `target < ip_actual` (salto hacia atrás, es decir, retorno de bucle):

1. `hot_counts[target]` se incrementa
2. Cuando `hot_counts[target] >= 50`, se inicia un nuevo `Trace` con `start_ip = target`
3. Grabación de rastreo está protegida: comienza solo si `trace_cache[target].is_none()` (sin rastreo compilado) e `is_recording` es falso

---

## Grabación de Rastreo

Cuando `is_recording` es verdadero, el intérprete ejecuta cada opcode normalmente **y** registra un `TraceOp` especializado para los tipos de tiempo de ejecución actuales.

### ¿Qué es Rastreable?

| Rastreable | No Rastreable |
|---|---|
| Aritmética Int/Float | Métodos de objeto (excepto ArraySize, ArrayGet, ArrayPush, etc.) |
| Comparaciones Int/Float | Operaciones de cadena |
| Lógica booleana | Llamadas de función |
| Acceso/modificación global | E/S, HTTP |
| Incrementos de contador | Operaciones de fibra |
| Operaciones de matriz: tamaño, obtener, empujar, actualizar | |
| Operaciones de conjunto: tamaño, contiene | |

---

## Compilación de Rastreo

Cuando el rastreo retorna a `start_ip`, el `Trace` completo se pasa a `JIT::compile()` en Cranelift. Cranelift compila la secuencia de `TraceOp` en una función nativa con la firma `JITFunction` (3 parámetros, retorno `i32`).

---

## Ejecución de Rastreo

En cada iteración de despacho del bucle, antes de obtener el siguiente opcode:

Si hay una función nativa compilada disponible (`native_ptr != null`), se llama directamente via `transmute` — omitiendo completamente el intérprete para todo el cuerpo del bucle.

---

## JIT de Método — Compilación de Función

Las funciones **sin bucles** (`has_loops = false`) y con menos de 500 instrucciones de bytecode son elegibles para compilación JIT de Método.

### Desencadenante

Después de **10 llamadas** a la misma función, se dispara la compilación JIT de forma asincrónica.

---

## Variantes de TraceOp

### Datos e Movimiento

| Variante | Descripción |
|---|---|
| `LoadConst { dst, val }` | Cargar constante en registro |
| `Move { dst, src }` | Copiar registro |
| `GetVar { dst, idx }` | Cargar global |
| `SetVar { idx, src }` | Almacenar global |

### Aritmética Entera

| Variante | Descripción |
|---|---|
| `AddInt / SubInt / MulInt` | Operaciones básicas (desbordamiento envolvente) |
| `DivInt / ModInt` | Con `fail_ip` para división por cero / desbordamiento |
| `PowInt` | Exponenciación via función C externa |
| `IntConcat` | Concatenación de dígitos (123 ++ 456 = 123456) |

---

## Configuración de JIT

El compilador Cranelift utiliza la configuración:
- `opt_level`: "speed"
- `use_colocated_libcalls`: "false"
- `is_pic`: "false"
- `regalloc_checker`: "false"
