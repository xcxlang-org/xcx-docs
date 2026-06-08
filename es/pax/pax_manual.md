# Manual del Gestor de Paquetes PAX

PAX es el gestor de paquetes oficial de XCX, integrado directamente en el binario `xcx`. Gestiona dependencias, scaffolding de proyectos, publicación en el registro y automatización de compilación.

---

## Configuración del proyecto: `project.pax`

Todo proyecto PAX debe tener un archivo `project.pax` en su directorio raíz. Utiliza un formato declarativo personalizado.

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

### Campos

| Campo         | Requerido | Descripción                                                                  |
|---------------|-----------|------------------------------------------------------------------------------|
| `name`        | Sí        | Nombre lógico del proyecto/paquete.                                          |
| `version`     | Sí        | Cadena de versión semántica (ej. `"1.0.0"`).                                 |
| `author`      | No        | Nombre del autor, visible en el registro.                                    |
| `description` | No        | Descripción corta del proyecto.                                              |
| `main`        | No        | Punto de entrada personalizado. Por defecto `src/main.xcx` si se omite.     |
| `tags`        | No        | Lista de etiquetas para la descubribilidad en el registro.                   |
| `files`       | No        | Lista explícita de archivos a incluir al publicar. Si se omite, PAX excluye archivos no-código automáticamente. |
| `deps`        | No        | Lista de dependencias. Soporta nombres versionados, atajos `author/repo` y URLs directas. |

### Formatos de dependencia

```pax
deps :: [
    "mylib@1.0.0",           --- paquete del registro con versión específica
    "mylib@latest",          --- última versión del registro
    "author/mylib@2.1.0",    --- con prefijo de autor
    "https://example.com/lib.xcx"  --- URL directa
]
```

---

## Archivo de bloqueo: `pax.lock`

PAX mantiene automáticamente un archivo `pax.lock` después de cada instalación. Registra la versión resuelta exacta y la ruta local de cada dependencia instalada.

```json
{
  "mylib": { "version": "mylib@1.0.0", "file": "lib/mylib/" },
  "otherlib": { "version": "otherlib@latest", "file": "lib/otherlib/" }
}
```

- **No** edites `pax.lock` manualmente.
- Confírmalo en el control de versiones para compilaciones reproducibles.

---

## Estructura de directorios

Un proyecto PAX estándar sigue esta estructura:

```
my_project/
├── project.pax       # Configuración del proyecto
├── pax.lock          # Archivo de bloqueo autogenerado
├── src/
│   └── main.xcx      # Punto de entrada principal
├── lib/              # Dependencias instaladas (gestionadas por PAX)
└── tests/            # Tests específicos del proyecto
```

---

## Referencia de comandos

PAX se invoca mediante `xcx pax <command>`.

| Comando                      | Descripción                                            |
|------------------------------|--------------------------------------------------------|
| `xcx pax new <name>`         | Crear una nueva estructura de proyecto.                |
| `xcx pax install`            | Descargar todas las dependencias en `lib/`.            |
| `xcx pax add <dep>`          | Añadir una dependencia e instalarla inmediatamente.    |
| `xcx pax remove <name>`      | Eliminar una dependencia de `project.pax`.             |
| `xcx pax clone <package>`    | Descargar un paquete publicado como proyecto local.    |
| `xcx pax run [path]`         | Ejecutar el punto de entrada del proyecto.             |
| `xcx pax search <query>`     | Buscar paquetes disponibles en el registro.            |
| `xcx pax login <token>`      | Almacenar un token de autenticación del registro.      |
| `xcx pax logout`             | Eliminar el token de autenticación almacenado.         |
| `xcx pax whoami`             | Verificar la cuenta actual del registro.               |
| `xcx pax publish`            | Publicar el proyecto en el registro PAX.               |

---

## Detalles de los comandos

### `xcx pax new <name>`

Crea un nuevo proyecto con la estructura de directorios estándar.

```sh
xcx pax new my_project
```

Crea:
```
my_project/
├── project.pax
├── src/
│   └── main.xcx
└── README.md
```

---

### `xcx pax install`

Lee `project.pax`, resuelve todas las `deps` y las descarga en `lib/`. Omite los paquetes ya presentes (por nombre).

```sh
xcx pax install
```

- Solo instala los archivos declarados en el campo `files` de la dependencia.
- Si la dependencia no tiene campo `files`, PAX excluye automáticamente `README.md`, `LICENSE`, `tests/`, `docs/`, `.git*` y `project.pax`.
- Actualiza `pax.lock` tras cada descarga exitosa.

---

### `xcx pax add <dep>`

Añade una dependencia a `project.pax` y ejecuta la instalación inmediatamente.

```sh
xcx pax add mylib@1.0.0
xcx pax add author/mylib@latest
```

---

### `xcx pax remove <name>`

Elimina una entrada de dependencia de `project.pax`.

```sh
xcx pax remove mylib
```

> **Nota:** Las carpetas `lib/` y las entradas de `pax.lock` no se limpian automáticamente. Vuelve a ejecutar `xcx pax install` o elimínalas manualmente.

---

### `xcx pax clone <package>`

Descarga un paquete publicado del registro PAX como un proyecto local completo, listo para ejecutar o modificar.

```sh
xcx pax clone snake_game
xcx pax clone author/snake_game
```

- Crea un directorio con el nombre del paquete en el directorio de trabajo actual.
- Copia la estructura completa del proyecto incluyendo `project.pax`, `src/` y todos los archivos declarados.
- **No** instala las dependencias automáticamente.

**Flujo de trabajo típico:**
```sh
xcx pax clone snake_game
cd snake_game
xcx pax install
xcx pax run
```

---

### `xcx pax run [path]`

Ejecuta el proyecto. Orden de resolución del punto de entrada:

1. Ruta proporcionada como argumento (ej. `xcx pax run other/main.xcx`)
2. Campo `main` en `project.pax`
3. Por defecto: `src/main.xcx`

```sh
xcx pax run
xcx pax run src/alt.xcx
```

---

### `xcx pax search <query>`

Busca en el registro PAX paquetes que coincidan con la consulta.

```sh
xcx pax search json
```

Formato de salida:
```
- json_utils (v1.2.0) by alice
- fast_json (v0.9.1) by bob
```

---

### `xcx pax login <token>`

Almacena un token de autenticación del registro localmente en `.pax_token`.

```sh
xcx pax login your_token_here
```

---

### `xcx pax logout`

Elimina el token de autenticación almacenado.

```sh
xcx pax logout
```

---

### `xcx pax whoami`

Verifica la sesión actual del registro e imprime la información de la cuenta.

```sh
xcx pax whoami
# Account: alice [developer]
```

---

### `xcx pax publish`

Publica el proyecto actual en el registro PAX. Requiere inicio de sesión.

```sh
xcx pax publish
```

- Lee los metadatos de `project.pax` (`name`, `version`, `description`).
- Escanea y sube todos los archivos del proyecto (excluyendo `.zip`, `.pax_token`, `.git`).
- Envía un archivo comprimido y un manifiesto de archivos al registro.

> **Nota:** Actualiza `version` en `project.pax` antes de cada publicación. El registro puede rechazar versiones duplicadas.

---

## Configuración del registro

Por defecto PAX se conecta a `pax.xcxlang.com`. Puedes sobreescribir esto con un archivo `.pax_config` en el directorio raíz del proyecto:

```json
{
  "registry": "https://my-custom-registry.example.com"
}
```

- HTTP se usa automáticamente para `localhost` y `127.0.0.1`.
- HTTPS se usa para todos los demás hosts.
