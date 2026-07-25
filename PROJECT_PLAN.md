# TAC MatchPhoto — Plan del Proyecto

> Estado: **Fase 0 — Fundamentos y documentación de arquitectura.**
> Este documento resume el alcance, las decisiones tecnológicas, las dependencias, los riesgos y los hitos de la primera fase del proyecto. No contiene código; ver `DEVELOPMENT_PHASES.md` para el desglose de implementación.

## 1. Resumen ejecutivo

TAC MatchPhoto es una aplicación de escritorio para Windows, de un único usuario (una fotógrafa de fútbol base), destinada a gestionar el flujo completo de producción fotográfica de partidos: organización de temporadas/clubes/equipos/jugadores/partidos, importación masiva de fotografías (JPG/JPEG/NEF) desde tarjetas o discos, lectura de metadatos EXIF, generación de miniaturas/previsualizaciones, revisión rápida con valoración y selección, agrupación de ráfagas, y exportación final con marca de agua.

Volumen de referencia: ~500 fotografías JPG+NEF por partido, múltiples partidos por temporada, múltiples temporadas. Todo el procesamiento y almacenamiento es **100% local**, sin nube ni APIs externas.

## 2. Alcance de esta fase (MVP)

**Incluido:**

1. Gestión de temporadas, clubes, categorías, equipos, jugadores (con historial jugador-equipo-temporada) y partidos.
2. Importación desde tarjeta/carpeta/disco externo, en modo copia o modo vínculo (sin duplicar).
3. Lectura de EXIF y extracción de previsualización embebida en NEF.
4. Generación y almacenamiento de miniaturas y previsualizaciones en caché local.
5. Modo Revisión Deportiva: navegación rápida, estados, valoración por estrellas, filtros, agrupación de ráfagas.
6. Exportación a JPG con marca de agua, sin alterar originales.
7. Base de datos SQLite con capa de acceso a datos y migraciones versionadas.
8. Copias de seguridad automáticas de la base de datos.
9. Detección de duplicados por huella digital (SHA-256).
10. Aviso de fuente de importación no disponible (disco externo desconectado).
11. Tablas vacías, sin lógica, preparadas para reconocimiento facial y funciones futuras.

**Explícitamente fuera de alcance en esta fase:**

- Reconocimiento facial, detección de dorsales, clasificación de jugadas.
- Cualquier forma de IA generativa o retoque automático de imagen.
- Revelado RAW completo (demosaico); solo se usa la previsualización JPEG embebida en el NEF.
- Sincronización en la nube, cuentas de usuario múltiples, colaboración remota.
- Interfaz de usuario final pulida al 100% (se construye por fases, ver `DEVELOPMENT_PHASES.md`).

## 3. Pila tecnológica

| Capa | Tecnología | Motivo |
|---|---|---|
| Shell de escritorio | Tauri 2 | Binarios ligeros, acceso nativo a sistema de archivos, sin runtime de nube, buen soporte Windows |
| Interfaz | React + TypeScript estricto + Vite | Ecosistema maduro, tipado fuerte, HMR rápido en desarrollo |
| Lógica de sistema de archivos, hashing, EXIF, miniaturas | Rust (dentro de `src-tauri`) | Rendimiento, seguridad de memoria, acceso directo a E/S de disco |
| Base de datos | SQLite (embebida, `rusqlite` con SQLite enlazado/"bundled") | Cero configuración, un único archivo portable, adecuada para uso local de un solo usuario |
| Migraciones | `refinery` (Rust) sobre archivos `.sql` versionados | Migraciones reproducibles y auditable en control de versiones |
| Metadatos EXIF / preview NEF | `exiftool` como binario adjunto ("sidecar") de Tauri | Ver sección de riesgos y `ARCHITECTURE.md §5` para la justificación completa |

Ver `ARCHITECTURE.md` para el detalle de capas y `DATABASE_SCHEMA.md` para el esquema completo.

## 4. Estructura de carpetas del repositorio

```
tac-matchphoto/
├── PROJECT_PLAN.md
├── ARCHITECTURE.md
├── FACE_PROTECTION_POLICY.md
├── DATABASE_SCHEMA.md
├── DEVELOPMENT_PHASES.md
├── package.json
├── tsconfig.json
├── vite.config.ts
├── .eslintrc.cjs / eslint.config.js
├── .prettierrc
├── src/                          # Frontend (React + TS)
│   ├── app/                      # Shell de la app, layout, router
│   ├── screens/                  # Una carpeta por pantalla (Inicio, Temporadas, ...)
│   ├── features/                 # Lógica de UI por dominio (import, review, export, catalog)
│   ├── domain/                   # Tipos y modelos de dominio TS (sin dependencias de UI)
│   ├── state/                    # Stores tipados (Zustand), un slice por dominio
│   ├── ipc/                      # Wrappers tipados sobre `invoke()` de Tauri, un módulo por comando Rust
│   ├── components/                # Componentes de UI reutilizables (design system interno)
│   ├── hooks/
│   ├── styles/
│   └── test/                     # Configuración de Vitest, utilidades de test
├── src-tauri/                    # Backend (Rust)
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   ├── migrations/               # Archivos .sql versionados (refinery)
│   ├── binaries/                 # exiftool.exe (sidecar) por plataforma
│   └── src/
│       ├── main.rs
│       ├── commands/             # Comandos Tauri expuestos al frontend, por dominio
│       ├── db/                   # Conexión, pool, migraciones, repositorios
│       ├── domain/                # Structs de dominio compartidos por los repositorios
│       ├── import/                # Pipeline de importación: hashing, EXIF, miniaturas, pares RAW+JPG
│       ├── fs_utils/               # Sanitización de rutas, layout de biblioteca, disponibilidad de discos
│       ├── export/                 # Composición de marca de agua, escritura de exportaciones
│       ├── backup/                 # Copias de seguridad de la base de datos
│       └── logging/                # Registro local (tracing)
└── docs/
    └── adr/                       # Registros de decisiones de arquitectura futuras (opcional)
```

## 5. Estructura física de datos de usuario (fuera del repositorio de código)

Definida en detalle en `ARCHITECTURE.md §4`. Resumen:

```
TAC_MATCHPHOTO/
├── Biblioteca/
│   └── <temporada>/<club>/<equipo>/<fecha_equipo_vs_rival>/{RAW,JPG,PREVIEWS,EXPORTS}
├── BaseDatos/
├── Cache/                # Miniaturas y previsualizaciones regenerables
├── MarcasDeAgua/
└── CopiasBaseDatos/
```

## 6. Dependencias propuestas y justificación

### 6.1 Frontend (TypeScript/React)

| Paquete | Uso |
|---|---|
| `react`, `react-dom` | Base de la interfaz |
| `react-router-dom` | Enrutado entre las 11 pantallas iniciales |
| `zustand` | Estado global tipado, ligero, sin boilerplate de reducers |
| `@tanstack/react-virtual` | Virtualización de la tira de miniaturas y listas largas (500+ fotos por partido) |
| `@tauri-apps/api` | Puente tipado hacia comandos y eventos de Tauri |
| `@tauri-apps/plugin-dialog` | Selector nativo de carpeta/tarjeta/disco para importación |
| `@tauri-apps/plugin-fs` | Acceso controlado a sistema de archivos desde el frontend cuando aplique |
| `date-fns` | Formateo/cálculo de fechas (temporadas, partidos) sin dependencias pesadas |
| `zod` | Validación de formularios y de datos que cruzan el límite IPC |
| `vite`, `typescript` | Build y tipado |
| `vitest`, `@testing-library/react` | Pruebas unitarias de frontend |
| `eslint`, `prettier`, `typescript-eslint` | Calidad y consistencia de código |

### 6.2 Backend (Rust, `src-tauri`)

| Crate | Uso |
|---|---|
| `tauri` (v2) | Runtime de la aplicación de escritorio |
| `rusqlite` (feature `bundled`) | Acceso a SQLite sin depender de una librería del sistema en Windows |
| `r2d2`, `r2d2_sqlite` | Pool de conexiones para acceso concurrente desde comandos async |
| `refinery` | Migraciones SQL versionadas y reproducibles |
| `serde`, `serde_json` | Serialización de datos entre Rust y el frontend, y de EXIF crudo |
| `sha2` | Cálculo de huella digital SHA-256 (deduplicación) |
| `image` | Decodificación/redimensionado/codificación de miniaturas y previsualizaciones JPG |
| `fast_image_resize` | Redimensionado de imágenes de alto rendimiento (SIMD) para lotes de cientos de fotos |
| `kamadak-exif` | Lectura rápida de EXIF en JPG sin lanzar un proceso externo (complementa a `exiftool` en el caso simple) |
| `walkdir` | Recorrido de carpetas/tarjetas durante la importación |
| `chrono` | Manejo de fechas/horas (EXIF, partidos, temporadas) |
| `tracing`, `tracing-subscriber`, `tracing-appender` | Registro local persistente (logging) |
| `thiserror` | Tipos de error de dominio explícitos y ergonómicos |
| `uuid` | Identificadores para sesiones de importación/exportación cuando corresponda |
| `sysinfo` o API de Windows (`windows` crate, `GetVolumeInformationW`) | Detección de volúmenes/discos externos y su disponibilidad |

### 6.3 Binario externo empaquetado (sidecar de Tauri)

| Herramienta | Uso |
|---|---|
| `exiftool` (empaquetado como sidecar, ejecución 100% local) | Lectura robusta de EXIF de JPG y NEF, y extracción de la previsualización JPEG embebida en archivos NEF. Ver justificación detallada en `ARCHITECTURE.md §5`. |

No se añade ninguna dependencia de red, SDK de nube, ni servicio externo. Todas las dependencias anteriores operan 100% localmente y sin conexión.

## 7. Riesgos técnicos identificados

### 7.1 RAW / NEF

- **Riesgo:** No existe una única librería Rust "pura" que extraiga de forma fiable la previsualización embebida en NEF para todos los cuerpos y firmwares Nikon (los offsets de MakerNote varían).
  **Mitigación:** usar `exiftool` (herramienta madura, con base de datos de tags actualizada activamente y soporte extenso de Nikon) como sidecar local para la extracción de preview y EXIF completo; usarlo en modo `-stay_open` para evitar el coste de arrancar un proceso por cada uno de los ~500 archivos de un partido.
- **Riesgo:** Tamaño de archivo NEF (20–45 MB) hace costoso el hashing y la copia si no se hace en streaming.
  **Mitigación:** lectura en bloques (buffered) para SHA-256 y copia; operaciones de importación en hilos en segundo plano, con reporte de progreso por evento Tauri.
- **Riesgo futuro (documentado, no resuelto en esta fase):** un revelado RAW completo (demosaico) requeriría una librería adicional (p. ej. `rawloader` + `imagepipe`) y mayor cómputo; queda fuera del MVP por decisión explícita del alcance.

### 7.2 Rendimiento

- **Riesgo:** Generar miniaturas/previews para ~500 fotos por partido (algunas sesiones pueden acumular varios miles de archivos por temporada) puede ser lento si se hace de forma síncrona o secuencial ingenua.
  **Mitigación:** pipeline de importación en cola con paralelismo acotado (worker pool), generación de miniatura y preview desacoplada de la inserción en base de datos (la fila de la foto se crea primero con estado "procesando" y se actualiza al terminar), progreso incremental vía eventos Tauri.
- **Riesgo:** Renderizar una tira de 500+ miniaturas sin virtualización congelaría la interfaz.
  **Mitigación:** `@tanstack/react-virtual` para la tira, y precarga controlada (solo foto siguiente/anterior) en el visor principal, no de todo el partido.
- **Riesgo:** Escrituras SQLite concurrentes durante importación masiva.
  **Mitigación:** modo WAL de SQLite, pool de conexiones (`r2d2_sqlite`), transacciones por lote en vez de una transacción por archivo.

### 7.3 Discos externos / rutas vinculadas

- **Riesgo:** En modo "vínculo" (sin copiar), si el disco externo o la tarjeta se desconecta, desaparece o cambia de letra de unidad (comportamiento habitual en Windows), las fotos referenciadas quedan inaccesibles y el usuario debe enterarse claramente.
  **Mitigación:** guardar en `import_sources` el número de serie de volumen (no solo la letra de unidad) para poder reconciliar si la letra cambia; verificación de disponibilidad al iniciar la app y bajo demanda; marcar `source_available = false` en las fotos afectadas y mostrar un aviso visible en la interfaz en vez de fallar silenciosamente.
- **Riesgo:** Rutas de biblioteca muy largas (`temporada/club/equipo/fecha_equipo_vs_rival/RAW/nombre.NEF`) pueden acercarse al límite clásico de 260 caracteres de Windows.
  **Mitigación:** sanitización y acortamiento controlado de componentes de carpeta, y uso de prefijos de ruta extendida (`\\?\`) al operar sobre el sistema de archivos desde Rust cuando sea necesario.

### 7.4 Integridad de datos

- **Riesgo:** Una importación interrumpida (cancelación, corte de energía, extracción de la tarjeta) podría dejar la base de datos en un estado inconsistente.
  **Mitigación:** cada archivo se procesa e inserta dentro de una transacción atómica propia; la sesión de importación (`import_sessions`) registra qué se completó; una cancelación detiene la cola sin revertir lo ya confirmado y sin dejar filas a medio insertar.

## 8. Hitos de esta fase

Ver el desglose completo en `DEVELOPMENT_PHASES.md`. Resumen de macro-hitos:

1. **Fundamentos** — repositorio, esquema SQLite + migraciones, capa de acceso a datos, logging, shell de la app.
2. **Catálogo** — CRUD de temporadas, clubes, categorías, equipos, jugadores, partidos.
3. **Importación** — copia/vínculo, hashing, EXIF, miniaturas/previews, detección de pares RAW+JPG.
4. **Revisión Deportiva** — visor, estados, valoración, ráfagas, atajos de teclado.
5. **Exportación** — marca de agua, generación de JPG de salida.
6. **Configuración y mantenimiento** — copias de seguridad, gestión de biblioteca, registro de actividad.
7. **Endurecimiento** — pruebas unitarias de funciones críticas, empaquetado Windows.

## 9. Restricción permanente de arquitectura

Esta aplicación, en cualquier fase presente o futura, **nunca modificará rostros humanos**. Esta regla es una restricción de arquitectura permanente, no una preferencia de producto. Ver `FACE_PROTECTION_POLICY.md` para el detalle completo, que debe leerse antes de implementar cualquier función relacionada con personas, reconocimiento o edición de imagen.
