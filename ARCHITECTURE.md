# TAC MatchPhoto — Arquitectura

## 1. Principios rectores

1. **Los originales son sagrados.** Ninguna operación de la aplicación escribe, sobrescribe ni mueve un archivo original importado. Las únicas escrituras sobre "el original" son metadatos en SQLite (nunca en el propio archivo de imagen).
2. **Todo es local.** Sin llamadas de red, sin SDKs de nube, sin telemetría externa. La base de datos, la caché, las miniaturas y las exportaciones viven en el disco del usuario.
3. **No destructivo por diseño.** Los archivos nuevos (miniaturas, previews, exportaciones, marcas de agua) se crean siempre en rutas distintas a las de origen; nunca se genera un archivo nuevo con el mismo nombre/ruta que un original.
4. **Los rostros son intocables.** Restricción de arquitectura permanente — ver `FACE_PROTECTION_POLICY.md`. Ninguna capa de esta arquitectura debe exponer una operación capaz de regenerar o alterar píxeles de un rostro.
5. **Separación estricta de capas.** Interfaz, dominio, persistencia y sistema de archivos no se mezclan. El frontend no sabe cómo se calcula un hash ni dónde vive físicamente un archivo; solo conoce tipos de dominio e invoca comandos tipados.
6. **Preparada para crecer, sin construir de más.** Las tablas y los espacios de nombres de comandos para reconocimiento facial y funciones futuras existen desde ahora (vacíos, sin lógica) para no requerir migraciones destructivas más adelante.

## 2. Vista general de capas

```
┌─────────────────────────────────────────────────────────────────┐
│  React (src/)                                                    │
│  screens/  →  features/  →  components/                          │
│      │             │                                              │
│      ▼             ▼                                              │
│  state/ (Zustand, tipado por dominio)                             │
│      │                                                             │
│      ▼                                                             │
│  ipc/ (wrappers tipados sobre invoke() y listen())                │
└───────────────────────────┬────────────────────────────────────┘
                             │  Tauri IPC (JSON tipado, validado con zod en el borde)
┌───────────────────────────▼────────────────────────────────────┐
│  Rust (src-tauri/src/)                                           │
│  commands/  →  valida entrada, orquesta, no contiene SQL          │
│      │                                                             │
│      ├──► import/     (pipeline de importación)                   │
│      ├──► export/     (composición de marca de agua)              │
│      ├──► fs_utils/   (sanitización de rutas, discos, layout)      │
│      ├──► backup/     (copias de seguridad de la base de datos)    │
│      └──► domain/     (structs y reglas de dominio)                │
│                │                                                    │
│                ▼                                                    │
│  db/ (repositorios por entidad, un módulo por tabla/agregado)       │
│                │                                                    │
│                ▼                                                    │
│  SQLite (BaseDatos/tac_matchphoto.db, modo WAL)                     │
└───────────────────────────────────────────────────────────────────┘
```

**Regla de dependencia:** las flechas solo pueden apuntar hacia abajo. `commands/` puede llamar a `import/`, `export/`, `db/`, `fs_utils/`; ningún módulo de `db/` conoce Tauri; ningún módulo de `domain/` conoce SQL ni HTTP ni UI.

## 3. Frontend: organización y estado

- **`domain/`** — tipos TypeScript puros (interfaces, enums de estado, value objects). Sin `any`. Reflejan los tipos de dominio de Rust (mismos nombres de campo en `camelCase` en TS vs `snake_case` en Rust/SQLite, mapeados explícitamente en `ipc/`).
- **`ipc/`** — un módulo por dominio de comandos (`ipc/seasons.ts`, `ipc/import.ts`, `ipc/review.ts`, `ipc/export.ts`, ...). Cada función tiene firma tipada de entrada/salida y es el único lugar del frontend que llama a `invoke()`. Valida la forma de la respuesta con `zod` antes de devolverla al resto de la app.
- **`state/`** — un store Zustand por dominio funcional (catálogo, importación, revisión, exportación, configuración/UI). Los stores no llaman a `invoke()` directamente: llaman a `ipc/`. Esto mantiene el estado testeable sin un backend real.
- **`features/`** — lógica de UI específica de un flujo (p. ej. `features/import/useImportQueue.ts`, `features/review/useKeyboardShortcuts.ts`).
- **`screens/`** — una carpeta por pantalla de las 11 iniciales, compuestas a partir de `features/` y `components/`. Las pantallas no contienen lógica de negocio.
- **`components/`** — sistema de componentes visual, oscuro, orientado a fotografía (ver `PROJECT_PLAN.md` y la guía de diseño visual). Sin conocimiento de dominio específico.

## 4. Backend Rust: organización

- **`commands/`** — la única superficie que Tauri expone al frontend (`#[tauri::command]`). Cada función: valida entrada, delega a `import/`, `export/`, `db/` o `fs_utils/`, traduce errores de dominio a errores serializables, nunca contiene una consulta SQL embebida directamente.
- **`db/`** —
  - `db/connection.rs`: apertura de la base de datos, configuración de `PRAGMA journal_mode=WAL`, `PRAGMA foreign_keys=ON`, pool `r2d2`.
  - `db/migrations/` + `../migrations/*.sql`: migraciones versionadas aplicadas con `refinery` al arrancar la app.
  - `db/repositories/`: un archivo por agregado (`seasons.rs`, `clubs.rs`, `teams.rs`, `players.rs`, `matches.rs`, `photos.rs`, `import_sessions.rs`, `burst_groups.rs`, `exports.rs`, `watermarks.rs`, `settings.rs`, `activity_log.rs`). Cada repositorio expone funciones de dominio (`create`, `get_by_id`, `list_by_match`, ...), nunca SQL crudo fuera de su propio archivo.
- **`import/`** — pipeline de importación (detallado en §6).
- **`export/`** — composición de marca de agua sobre copias renderizadas para exportación (nunca sobre el original).
- **`fs_utils/`** — sanitización de nombres de carpeta/archivo, construcción de rutas de biblioteca, detección de disponibilidad de volúmenes (número de serie de volumen en Windows), utilidades de ruta extendida (`\\?\`) para evitar el límite de 260 caracteres.
- **`backup/`** — copias de seguridad de la base de datos mediante `VACUUM INTO` (seguro en caliente, atómico), con rotación de copias antiguas.
- **`logging/`** — `tracing` configurado para escribir a `TAC_MATCHPHOTO/BaseDatos/logs/` con rotación diaria.

## 5. Decisión de diseño: EXIF y previsualización de NEF

**Contexto.** Un archivo NEF es un contenedor TIFF/EP. Nikon embebe habitualmente una previsualización JPEG de alta resolución (y una miniatura pequeña) dentro de sub-IFDs, con offsets que varían según cuerpo de cámara y versión de firmware. No se requiere en esta fase un revelado RAW (demosaico) completo — solo extraer esa previsualización embebida y los metadatos EXIF/MakerNote.

**Opciones evaluadas:**

| Opción | Evaluación |
|---|---|
| `kamadak-exif` (Rust puro) | Buen lector de tags EXIF/TIFF estándar; no ofrece de forma directa y fiable la extracción genérica de la mejor previsualización embebida en todos los MakerNotes de Nikon. Rápido y sin proceso externo — útil para JPG. |
| `rawloader` + `imagepipe` | Diseñado para decodificar el sensor RAW completo (demosaico), no solo para extraer la preview embebida; excede el alcance de esta fase ("no desarrollar un revelador RAW completo") y añade coste de cómputo innecesario para el MVP. |
| Escribir un parser TIFF/IFD propio | Reimplementaría lo que herramientas maduras ya resuelven; alto riesgo de fallos con modelos de cámara no probados; no se justifica para un equipo de una sola aplicación. |
| **`exiftool` como binario "sidecar" de Tauri** | **Elegida.** Herramienta madura, con base de tags Nikon mantenida activamente, capaz de extraer tanto el EXIF completo como la previsualización embebida (`-b -PreviewImage`, con `-b -JpgFromRaw` como respaldo) de forma fiable entre modelos y firmwares. Se ejecuta 100% localmente (no es un servicio de red), cumpliendo el requisito de "sin nube". Soporta modo `-stay_open true -@ -` para procesar cientos de archivos en un único proceso persistente, evitando el coste de arrancar un proceso por archivo. |

**Decisión final:** `exiftool` empaquetado como sidecar de Tauri es la fuente única de verdad para EXIF y previsualización de NEF (JPG y NEF por igual, para mantener una sola ruta de código). `kamadak-exif` queda disponible como dependencia de repuesto/verificación cruzada, sin ser parte crítica del pipeline en esta fase.

**Consecuencia arquitectónica:** el pipeline de importación depende de la disponibilidad del binario `exiftool.exe` empaquetado junto a la app (no requiere instalación por parte del usuario). Esto se documenta como requisito de empaquetado en `DEVELOPMENT_PHASES.md`.

## 6. Flujo de importación (alto nivel)

1. El usuario selecciona origen (tarjeta/carpeta/disco) y modo (copiar/vincular) en la pantalla **Importación**.
2. El frontend invoca `import_start` con la lista de rutas candidatas y el `match_id` destino.
3. Rust crea una fila en `import_sessions` (estado `running`) y encola cada archivo detectado (`.jpg`, `.jpeg`, `.nef`, sin distinguir mayúsculas/minúsculas).
4. Por archivo, en un worker en segundo plano:
   a. Leer nombre, tamaño, fechas del sistema de archivos.
   b. Calcular SHA-256 en streaming.
   c. Comprobar duplicado por hash contra `photos.file_hash_sha256`; si existe, registrar como omitido y continuar.
   d. Copiar o vincular según el modo elegido (si copia: construir ruta de biblioteca sanitizada y transferir; si vínculo: conservar la ruta original y registrar el volumen de origen).
   e. Invocar `exiftool` (proceso persistente) para EXIF y, si es NEF, para la previsualización embebida.
   f. Generar miniatura y previsualización con `image`/`fast_image_resize` a partir del JPG (directo) o de la previsualización extraída (NEF).
   g. Insertar la fila `photos` + `photo_exif` en una transacción atómica.
   h. Emitir evento de progreso al frontend (`import://progress`) con éxito/error de ese archivo.
5. Al finalizar el recorrido de archivos, ejecutar la detección de pares RAW+JPG por nombre base y carpeta, y poblar `photo_pairs`.
6. Cerrar la sesión de importación (`completed`, `cancelled` o `failed`) y registrar el resumen en `activity_log`.

**Cancelación:** una señal de cancelación detiene la cola tras el archivo en curso; nada queda a medio insertar porque cada archivo se confirma en su propia transacción; la sesión queda marcada `cancelled` con el recuento real de lo procesado.

## 7. Estructura física de la biblioteca

```
TAC_MATCHPHOTO/
├── Biblioteca/
│   └── <temporada>/<club>/<equipo>/<fecha>_<equipo>_vs_<rival>/
│       ├── RAW/
│       ├── JPG/
│       ├── PREVIEWS/
│       └── EXPORTS/
├── BaseDatos/
│   ├── tac_matchphoto.db
│   └── logs/
├── Cache/                  # Miniaturas y previews: regenerable, no crítico para backup
├── MarcasDeAgua/            # Imágenes de marca de agua reutilizables
└── CopiasBaseDatos/         # Backups rotados de tac_matchphoto.db
```

- Los nombres de carpeta se sanitizan (`fs_utils::sanitize_path_component`): se eliminan/city los caracteres inválidos en Windows (`< > : " / \ | ? *`), caracteres de control, espacios y puntos finales; se acortan de forma determinista si exceden un umbral razonable.
- En modo **vínculo**, no se crea copia en `Biblioteca/`; solo se referencia la ruta original y su volumen. `PREVIEWS/` y las miniaturas de caché sí se generan siempre localmente, independientemente del modo de importación, porque son artefactos derivados, no copias del original.

## 8. Exportación con marca de agua

- Entrada: selección de `photo_id`s + `watermark_id` (o marca de agua ad-hoc) + destino.
- El renderizador Rust (`export/`) carga el **mejor origen disponible** (el JPG original si existe, o la previsualización de mayor resolución si la selección es un NEF sin par JPG — limitación documentada del MVP al no existir revelado RAW completo), compone la marca de agua (imagen o texto, posición/opacidad configurables) y escribe un **archivo nuevo** en `.../EXPORTS/`.
- El original nunca se abre en modo escritura. El compositor de marca de agua es una operación de "overlay" sobre un lienzo nuevo; no existe, ni existirá, una ruta de código que module o reconstruya la imagen de origen — restricción compartida con `FACE_PROTECTION_POLICY.md`.
- Cada resultado se registra en `export_items` con su estado (`pending`/`done`/`error`).

## 9. Copias de seguridad

- Al iniciar y al cerrar la aplicación (y bajo demanda desde Configuración), se ejecuta `VACUUM INTO '<CopiasBaseDatos>/tac_matchphoto_<timestamp>.db'`, mecanismo atómico nativo de SQLite que no requiere bloquear la base de datos en uso.
- Rotación configurable (por defecto, conservar las últimas N copias) registrada en `db_backups`.

## 10. Disponibilidad de fuentes vinculadas

- `import_sources` guarda `root_path`, `volume_serial` (obtenido vía API de Windows) y `volume_label`.
- Al iniciar la app y bajo demanda, `fs_utils::check_source_availability` verifica cada fuente activa; si no se encuentra el volumen (por serie, no solo por letra de unidad, que puede cambiar), se marca `source_available = false` en las fotos vinculadas a esa fuente, y la interfaz debe mostrar un aviso explícito en vez de fallar silenciosamente al intentar mostrar la imagen.

## 11. Puntos de extensión preparados para el futuro

- **Reconocimiento facial:** tablas `detected_faces`, `person_profiles`, `face_embeddings`, `photo_people`, `recognition_confirmations` ya existen en el esquema (vacías, sin lógica). El futuro módulo de reconocimiento se integrará como un nuevo espacio de comandos (`commands/recognition/`) que **solo** escribe en estas tablas y **nunca** en la ruta de exportación/edición de imagen. Ver restricción permanente en `FACE_PROTECTION_POLICY.md`.
- **Detección de dorsales / clasificación de jugadas:** se anticipa como módulos adicionales de análisis que consumen `photos`/`photo_exif` en modo solo lectura y escriben en tablas propias aún no creadas; no requieren cambios en el pipeline no destructivo existente.
- **Editor futuro:** la tecla `E` se reserva desde ya en el modo Revisión Deportiva y no se asigna a ninguna función en esta fase.
- **Revelado RAW completo:** si se añade en el futuro, se integraría como un paso opcional adicional dentro de `import/` o como una acción bajo demanda en `export/`, sin alterar el modelo de "el original nunca se sobrescribe".

## 12. Logging y manejo de errores

- Errores de dominio tipados con `thiserror` en Rust; los comandos Tauri los traducen a un tipo de error serializable (`code`, `message`, `context`) consumido por el frontend.
- `tracing` registra en archivo local (rotado) los eventos relevantes: inicio/fin de importación, errores por archivo, exportaciones, copias de seguridad, cambios de disponibilidad de fuente.
- `activity_log` en SQLite es el registro de auditoría de negocio (qué se hizo, cuándo, sobre qué entidad), distinto del log técnico en archivo.

## 13. Pruebas

- **Rust:** pruebas unitarias junto a cada módulo (`#[cfg(test)]`) para: cálculo de hash, sanitización de rutas, detección de pares RAW+JPG, heurística de agrupación de ráfagas, aplicación de migraciones sobre una base de datos temporal, lógica de disponibilidad de fuente.
- **TypeScript:** Vitest para lógica de `state/` y `ipc/` (con mocks tipados de `invoke`), y para utilidades de dominio puras.
- Ninguna prueba borra, mueve ni modifica archivos reales del usuario: todas usan carpetas temporales (`tempfile` en Rust, directorios temporales del SO en TS) y datos de demostración.
