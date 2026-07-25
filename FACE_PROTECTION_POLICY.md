# TAC MatchPhoto — Política de Protección de Rostros

> **Estado: restricción permanente de arquitectura.**
> Este documento no es una guía de estilo ni una recomendación: es una regla de diseño obligatoria que se aplica a toda versión presente y futura de TAC MatchPhoto. Cualquier funcionalidad nueva, propia o de terceros, debe leerse y respetarse a la luz de este documento antes de ser implementada.

## 1. Principio fundamental

**Los rostros son intocables.**

TAC MatchPhoto gestiona fotografías de menores y adultos en el contexto de fútbol base. La aplicación puede **buscar, clasificar y organizar** fotografías en función de las personas que aparecen en ellas, pero **nunca puede alterar la representación física de un rostro** en ninguna fotografía, bajo ninguna circunstancia, modo, función experimental o solicitud de usuario.

## 2. Operaciones explícitamente prohibidas, para siempre

La aplicación, en ninguna fase presente o futura, podrá:

- Sustituir rostros (face swap).
- Regenerar rostros mediante modelos generativos.
- Cambiar expresiones faciales.
- Abrir ojos cerrados de forma artificial.
- Añadir sonrisas u otras expresiones no capturadas por la cámara.
- Embellecer o "mejorar" rasgos faciales.
- Rejuvenecer o envejecer personas.
- Cambiar rasgos faciales de cualquier tipo.
- Reconstruir o "rellenar" (inpainting) mediante IA cualquier parte de un rostro.
- Cualquier otra operación cuyo efecto sea modificar cómo se ve físicamente un rostro humano en una fotografía, más allá de ajustes globales de imagen no dirigidos a rostros específicos (ver §4).

Esta lista es orientativa, no exhaustiva: **cualquier operación cuyo propósito o efecto sea alterar la apariencia de un rostro queda prohibida por defecto**, exista o no en esta lista.

## 3. Uso permitido del reconocimiento facial (futuro)

Cuando se implemente en una fase posterior, el reconocimiento facial se usará **exclusivamente** para:

- Buscar fotografías donde aparece una persona concreta.
- Clasificar y agrupar fotografías por jugador/persona para facilitar la revisión y la entrega.
- Sugerir posibles coincidencias que un humano debe confirmar o descartar (`recognition_confirmations`).

El reconocimiento facial **no** es, ni será, una vía de entrada para ninguna operación de edición de imagen. Las tablas reservadas para esta función (`detected_faces`, `person_profiles`, `face_embeddings`, `photo_people`, `recognition_confirmations`, ver `DATABASE_SCHEMA.md`) almacenan únicamente coordenadas de detección, vectores de identidad y asociaciones persona-fotografía confirmadas por un humano — **nunca** datos generativos ni instrucciones de edición.

## 4. Qué sí está permitido (para no generar ambigüedad)

Para evitar interpretaciones erróneas, se aclara explícitamente que **no** están prohibidas por esta política las operaciones de imagen que no están dirigidas a modificar un rostro específico:

- Ajustes globales de exposición, contraste, balance de blancos o recorte de la fotografía completa (si se implementan en el futuro), siempre que se apliquen a la imagen entera y no de forma selectiva sobre un rostro.
- Superposición de una marca de agua sobre la fotografía completa (función de exportación de esta fase), que no altera el contenido de la imagen original ni reconstruye píxeles: añade una capa visible por encima.
- Recorte (crop) de la composición de una fotografía, siempre que no implique reconstruir contenido inexistente (eso sería inpainting, prohibido).

## 5. Cómo se aplica esta política a nivel de arquitectura

1. **Sin dependencias de edición generativa de imagen.** El proyecto no incluirá, en ninguna fase, una librería o servicio de generación/edición de imagen basada en IA (local o remota) en ninguna ruta de código que procese fotografías con personas.
2. **El pipeline de exportación es estrictamente de composición ("overlay"), nunca de reconstrucción.** Ver `ARCHITECTURE.md §8`. La única escritura de píxeles nuevos permitida en exportación es la marca de agua, sobre una copia, nunca sobre el original.
3. **El reconocimiento facial (futuro) es de solo lectura sobre la imagen.** Un detector de rostros y un extractor de embeddings leen la imagen para producir coordenadas y vectores; no producen ni modifican imagen.
4. **Revisión de código obligatoria.** Cualquier función nueva que reciba una imagen con rostros como entrada y produzca una imagen como salida debe justificar explícitamente, en su descripción de implementación, que no toca contenido facial, antes de integrarse.
5. **Sin excepciones por solicitud de usuario.** Aunque la aplicación es de un solo usuario y de confianza, esta restricción no es configurable ni desactivable desde ajustes: es una decisión de producto y de arquitectura, no una preferencia.

## 6. Procedimiento ante una propuesta que entre en conflicto con esta política

Si en cualquier fase futura se propone una función que pudiera interpretarse como una violación de esta política (por ejemplo, "mejora automática de fotos", "retoque con IA", "corrección de ojos cerrados"):

1. La función se detiene antes de implementarse.
2. Se contrasta explícitamente contra la lista de §2.
3. Si existe cualquier duda razonable de que la función toca contenido facial, se considera prohibida por defecto hasta que se demuestre lo contrario.
4. Solo el propietario del producto puede autorizar un cambio a este documento, y dicho cambio debe quedar registrado como una decisión explícita y fechada — nunca como un efecto colateral de otra tarea.

## 7. Relación con otros documentos

- `ARCHITECTURE.md §5, §8, §11` describe dónde se aplica esta política en el diseño técnico (extracción EXIF/preview, exportación, extensión futura de reconocimiento).
- `DATABASE_SCHEMA.md` describe la estructura vacía y sin lógica de las tablas de reconocimiento facial reservadas para el futuro.
- `PROJECT_PLAN.md §9` referencia esta política como restricción de alcance permanente.
