> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/language/errors_halt.md).

# XCX 3.1 Manejo de Errores (Halt)

XCX utiliza un sistema estructurado de `halt` para gestionar condiciones de tiempo de ejecución y errores.

## Niveles de Halt

| Nivel         | Comportamiento                                          | Caso de Uso                      |
|---------------|---------------------------------------------------|-------------------------------|
| `halt.alert`  | Imprime un mensaje; la ejecución continúa.            | Registro, advertencias no críticas|
| `halt.error`  | Imprime a stderr; aborta el marco actual.       | Errores de lógica recuperables      |
| `halt.fatal`  | Imprime un mensaje; termina la VM inmediatamente. | Fallo crítico, brecha de seguridad |

## Examples

```xcx
halt.alert >! "Cache missed, fetching from DB...";

if (divisor == 0) then;
    halt.error >! "¡División por cero!";
    return 0; --- La ejecución regresa al llamador desde el punto de recuperación
end;

if (NOT db.is_healthy()) then;
    halt.fatal >! "Base de datos corrupta. Apagado de emergencia.";
end;
```

## Pánico Semántico y en Tiempo de Ejecución

Ciertas operaciones inválidas resultan en un **pánico** automático (equivalente a `halt.fatal` o `halt.error` dependiendo del contexto):
- **División por cero** (aritmética)
- **Fallo de Análisis JSON**: Una cadena inválida en `json.parse()` resulta en una salida inmediata de la VM.
- **Travesía de Ruta**: Usar `..` en métodos `store` o rutas absolutas fuera de la raíz del proyecto dispara `halt.fatal`.
- **Profundidad de Recursión**: Exceder 800 marcos dispara `halt.error`.
