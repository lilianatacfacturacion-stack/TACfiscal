# TAC MatchPhoto — Fases de Desarrollo

> Desglose secuencial de implementación del MVP descrito en `PROJECT_PLAN.md`. Cada fase produce algo ejecutable y verificable antes de pasar a la siguiente. Ninguna fase se da por "hecha" sin haberse ejecutado y probado — no se declarará completada una fase por escrito sin evidencia de ejecución real.

## Fase 0 — Fundamentos del proyecto

**Objetivo:** repositorio inicializado, base de datos con esquema completo y migrable, capa de acceso a datos, logging, y una ventana Tauri vacía con la navegación lateral y las 11 pantallas como contenedores sin lógica.

- Inicializar proyecto Tauri 2 + React + TypeScript estricto + Vite.
- Configurar ESLint, Prettier, Vitest, `cargo fmt`/`cargo clippy`.
- Crear la estructura de carpetas descrita en `PROJECT_PLAN.md §4`.
- Implementar `db/connection.rs` (WAL, foreign keys, pool `r2d2`).
- Escribir `V1__init.sql` con el esquema completo de `DATABASE_SCHEMA.md`, integrar `refinery`.
- Implementar `fs_utils::sanitize_path_component` y utilidades de construcción de rutas de biblioteca (con pruebas unitarias).
- Implementar `backup::create_backup` (`VACUUM INTO`) y su rotación.
- Configurar `tracing` con rotación de logs locales.
- Shell de la app: navegación lateral, tema oscuro base, rutas para las 11 pantallas (sin funcionalidad).
- **Criterio de salida:** la app arranca en Windows (o en el entorno de desarrollo), crea `TAC_MATCHPHOTO/` y su base de datos vacía con el esquema aplicado, navega entre las 11 pantallas vacías. Pruebas unitarias de sanitización de rutas y de aplicación de migraciones en una base de datos temporal, ejecutadas y en verde.

## Fase 1 — Catálogo (Temporadas, Clubes, Categorías, Equipos, Jugadores, Partidos)

**Objetivo:** CRUD funcional completo de las pantallas 2 a 7.

- Repositorios Rust: `seasons`, `clubs`, `categories`, `teams`, `players`, `player_team_history`, `matches`, `photo_authorizations`.
- Comandos Tauri correspondientes, con validación de entrada.
- Capa `ipc/` tipada en el frontend, con validación `zod` en el borde.
- Stores Zustand de catálogo.
- Pantallas: Inicio (resumen), Temporadas, Clubes, Categorías y Equipos, Jugadores (con historial por equipo/temporada), Calendario y Partidos, Creación de Partido.
- **Criterio de salida:** posible crear una temporada, un club, una categoría, un equipo, un jugador con historial, y un partido, de principio a fin, con datos persistidos y recuperados tras reiniciar la app. Pruebas unitarias de los repositorios críticos (unicidad, integridad referencial).

## Fase 2 — Importación

**Objetivo:** pantalla 8 completamente funcional según `ARCHITECTURE.md §6`.

- Integrar `exiftool` como sidecar de Tauri (modo `-stay_open`).
- Implementar cálculo de SHA-256 en streaming y comprobación de duplicados.
- Implementar copia (con sanitización de rutas de biblioteca) y modo vínculo (registro de `import_sources` con volumen).
- Generación de miniatura y previsualización (`image` + `fast_image_resize`) desde JPG directo o desde preview extraída de NEF.
- Detección de pares RAW+JPG por nombre base y población de `photo_pairs`.
- `import_sessions` / `import_errors`, eventos de progreso Tauri, cancelación segura sin dejar filas a medio insertar.
- Pantalla de Importación: selección de origen (tarjeta/carpeta/disco), selección de modo, barra de progreso, listado de errores por archivo, botón de cancelar.
- **Criterio de salida:** importar un conjunto de prueba de fotos JPG+NEF (datos de demostración, no del catálogo real de la fotógrafa) desde una carpeta temporal, en ambos modos, con duplicados detectados correctamente, EXIF leído, miniaturas generadas, y cancelación verificada sin corrupción. Pruebas unitarias de hashing, detección de duplicados y detección de pares, ejecutadas sobre archivos de prueba en carpetas temporales.

## Fase 3 — Modo Revisión Deportiva

**Objetivo:** pantalla 9 completa, con foco en velocidad.

- Visor principal con precarga de imagen siguiente/anterior.
- Tira de miniaturas virtualizada (`@tanstack/react-virtual`).
- Panel de metadatos EXIF.
- Acciones de estado (seleccionar/destacar/descartar), valoración 1–5.
- Filtro por estado.
- Cálculo y visualización de grupos de ráfaga (heurística por cámara + proximidad temporal configurable, `app_settings.burst_group_gap_seconds`).
- Atajos de teclado: `→` siguiente, `←` anterior, `S` seleccionar, `D` descartar, `F` destacar, `1`–`5` valoración, `Espacio` pantalla completa, `G` ver grupo de ráfaga, `Esc` cerrar pantalla completa. La tecla `E` queda explícitamente sin asignar.
- **Criterio de salida:** revisar un partido de datos de demostración (varios cientos de fotos) con navegación fluida, sin recarga innecesaria de imágenes, atajos funcionando, y ráfagas visibles. Pruebas unitarias de la heurística de agrupación de ráfagas.

## Fase 4 — Exportación

**Objetivo:** pantalla 10 completa.

- CRUD de marcas de agua (`watermarks`), imagen o texto, posición/opacidad/escala configurables.
- Pipeline de exportación (`export/`): composición sobre copia nueva, nunca sobre el original; selección del mejor origen disponible por foto.
- `export_jobs` / `export_items` con estado por fotografía.
- Pantalla de Exportación: selección de fotos por estado/valoración, elección de marca de agua, destino, progreso.
- **Criterio de salida:** exportar una selección de fotos de demostración con marca de agua visible, verificando que los archivos originales permanecen sin modificar (comprobación de hash antes/después).

## Fase 5 — Configuración y mantenimiento

**Objetivo:** pantalla 11 completa.

- Configuración de ruta de biblioteca, modo de importación por defecto, retención de copias de seguridad, marca de agua por defecto.
- Gestión manual de copias de seguridad (crear/restaurar).
- Reverificación de disponibilidad de fuentes vinculadas, con aviso visible cuando una fuente no está disponible.
- Visor del registro de actividad (`activity_log`).
- **Criterio de salida:** cambiar configuración persiste correctamente; desconectar una fuente vinculada (simulada) produce el aviso esperado en la interfaz.

## Fase 6 — Endurecimiento y empaquetado

**Objetivo:** MVP listo para uso real por la fotógrafa en Windows.

- Cobertura de pruebas unitarias de todas las funciones críticas identificadas en `ARCHITECTURE.md §13`.
- Revisión de manejo de errores en todos los comandos Tauri (sin `unwrap()` en rutas de comando).
- Prueba de rendimiento con un partido de ~500 fotografías JPG+NEF de demostración: importación, revisión y exportación dentro de tiempos razonables para uso interactivo.
- Empaquetado Windows con `tauri build` (instalador `.msi`/`.exe`), incluyendo el sidecar `exiftool.exe`.
- Revisión final de que ninguna ruta de código toca contenido facial, conforme a `FACE_PROTECTION_POLICY.md`.
- **Criterio de salida:** instalador Windows generado y ejecutado en limpio, flujo completo (catálogo → importación → revisión → exportación) verificado de principio a fin con datos de demostración.

## Fuera de estas fases (explícitamente pospuesto)

Reconocimiento facial, detección de dorsales, clasificación de jugadas, revelado RAW completo, IA generativa o retoque automático. Las tablas y puntos de extensión ya reservados (`ARCHITECTURE.md §11`, `DATABASE_SCHEMA.md §8`) permiten incorporarlos en fases futuras sin migraciones disruptivas ni cambios de arquitectura, siempre respetando `FACE_PROTECTION_POLICY.md`.
