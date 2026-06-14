# Manual del Gestor de Paquetes PAX

PAX es el gestor de paquetes oficial para XCX, integrado directamente en el binario `xcx`. Gestiona dependencias, scaffolding de proyectos y automatización de builds.

## Configuración del Proyecto: `project.pax`

Todo proyecto PAX debe tener un archivo `project.pax` en su directorio raíz. Usa un formato declarativo personalizado.

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

- **name**: Nombre lógico del proyecto.
- **deps**: Lista de dependencias. Admite atajos de GitHub (`user/repo`) y URLs directas.

## Referencia de Comandos

PAX se invoca mediante `xcx pax <comando>`.

| Comando                  | Descripción                                                    |
|--------------------------|----------------------------------------------------------------|
| `xcx pax new <name>`     | Genera una nueva estructura de proyecto.                       |
| `xcx pax clone <package>`| Descarga un paquete publicado como proyecto local.             |
| `xcx pax install`        | Descarga las dependencias en el directorio `lib/`.             |
| `xcx pax add <dep>`      | Agrega una dependencia y la instala inmediatamente.            |
| `xcx pax remove <name>`  | Elimina una dependencia de `project.pax`.                      |
| `xcx pax search <query>` | Busca paquetes disponibles en el registro.                     |
| `xcx pax run [path]`     | Ejecuta el proyecto (entrada: `src/main.xcx`).                 |

### `xcx pax clone <package>`

Descarga un paquete publicado del registro PAX en un nuevo directorio local, listo para ejecutar o modificar.

```sh
xcx pax clone snake_game
xcx pax clone beta/snake_game
```

- El argumento es el nombre del paquete tal como aparece en el registro (opcionalmente con prefijo `author/`).
- Crea un directorio con el nombre del paquete en el directorio de trabajo actual.
- Copia la estructura completa del proyecto: `project.pax`, `src/`, y cualquier otro archivo declarado en `files`.
- **No** instala las dependencias automáticamente — ejecuta `xcx pax install` dentro del directorio clonado después.

```sh
xcx pax clone snake_game
cd snake_game
xcx pax install
xcx pax run
```

## Estructura de Directorios

Un proyecto PAX estándar sigue esta organización:
- `project.pax`: Configuración.
- `src/`: Código fuente (entrada principal: `main.xcx`).
- `lib/`: Dependencias descargadas (gestionadas por PAX).
- `tests/`: Pruebas específicas del proyecto.