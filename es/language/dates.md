> 📌 **Nota**: Esta documentación en español ha sido generada con IA y puede contener imprecisiones o errores. Para información más precisa, consulta la [documentación en inglés](../en/language/dates.md).

# XCX 3.1 Fechas y Hora

## Creación

Las fechas se crean usando la función `date()` o `date.now()`.

```xcx
date: d1 = date("2024-12-25");               --- YYYY-MM-DD
date: d2 = date("25/12/2024", "DD/MM/YYYY"); --- Custom format
date: now = date.now();                      --- Current system time
```

## Propiedades

Los objetos de fecha exponen campos enteros de solo lectura:

- `.year`
- `.month` (1-12)
- `.day` (1-31)
- `.hour` (0-23)
- `.minute` (0-59)
- `.second` (0-59)

## Aritmética y Comparación

Las fechas soportan suma/resta de días y comparación contra otras fechas.

```xcx
date: tomorrow = some_date + 1;
date: yesterday = some_date - 1;
i: days_between = christmas - halloween;  --- Resultado en días enteros
b: is_before = christmas < new_year;
```

## Formato

```xcx
s: s1 = now.format();                    --- Predeterminado: "YYYY-MM-DD HH:mm:ss"
s: s2 = now.format("DD/MM/YYYY HH:mm");  --- Salida personalizada
```

**Tokens:** `YYYY` (Año), `MM` (Mes 01-12), `DD` (Día 01-31), `HH` (Hora 00-23), `mm` (Minuto), `ss` (Segundo), `SSS` o `ms` (Milisegundos), `M` y `D` (Sin cero inicial).
