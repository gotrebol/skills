# CHANGELOG — Skill de Trébol

Historial de cambios al skill de Trébol distribuido vía [skills.sh](https://www.skills.sh). Las fechas corresponden al campo `last_updated` del frontmatter de `SKILL.md`.

Los integradores pueden preguntarle a su asistente con IA *"¿qué cambió en la última versión del skill de Trébol?"* y recibir un resumen consultando este archivo.

## 2026-08-12

- **`json_schema` en creación de procesos de extracción** (`POST /v2/custom-item-types/{id}/processes`): ahora se puede enviar `json_schema` al crear un proceso de extracción. Cuando se envía, `auto_improve` se establece en `false` automáticamente (enviar `auto_improve: true` junto con `json_schema` devuelve 400). Si no se envía `json_schema`, el comportamiento anterior se mantiene (`auto_improve` debe ser `true`). Misma regla aplica al actualizar (`PATCH`) un proceso con `json_schema`. La guía incluye nueva sección "Estructura del `json_schema`" con formato requerido, campos anulables (`"type": ["string", "null"]`) y tres ejemplos (plano, con arrays anidados, y mixto). Impacto en integraciones:
  - **Creación**: `POST` acepta `json_schema` opcional. Con `json_schema` → `auto_improve` forzado a `false`. Sin `json_schema` → `auto_improve` debe ser `true`.
  - **Actualización**: `PATCH` con `json_schema` → `auto_improve` forzado a `false`.
  - **Sin espera asíncrona**: al proveer `json_schema` propio no hay que esperar a que la mejora automática genere el esquema; el proceso queda listo de inmediato.
  - Spec y copia del skill sincronizadas con los nuevos ejemplos y errores.

## 2026-07-24

- **Corrección del nombre del campo en las respuestas de `doc_splitter`**: los cortes se leen bajo `item_value.split_documents` (con `has_multiple_documents` a la par), NO bajo `value.split_documents`. El schema `PublicVerificationItem` define `item_value` como el contenedor del payload por tipo de ítem (confirmado en `../business-verification/src/lib/verifications/getById.ts:234,873`). La guía, el ResponseExample y la nota de novedades quedaron corregidos.
- **Webhooks: `item_error` ampliado**: `verification_item.v2.completed`, `.internal_status_changed` y `.extraction_completed` ya no listan solo `password_protected_pdf` / `get_input_file_info_failed`. Ahora reflejan la lista real: comunes documentales + códigos específicos de `doc_splitter` y `doc_validation`, tratando el campo como `string` opaco. La doc apunta al schema `PublicVerificationItem.item_error` del OpenAPI como fuente única.
- **Ciclo de vida público de `doc_splitter` aclarado**: `split_success` / `split_failed` son estados **internos** (`internalStatus`), no públicos. La API expone solo `item_status: "pending" | "complete"`; el éxito o fallo se distingue por `item_error`. Regla para consumir cortes: `item_status === "complete"` **y** `item_error` es `null`. Un `add-items` con `file_source: "item"` apuntando a un `doc_splitter` fallado responde `409 source_item_not_ready`.
- **`item_error` de `doc_splitter` desglosado en la spec**: el OpenAPI ahora expone los 3 códigos granulares del backend (`pdf_slice_failed`, `pdf_slice_upload_failed`, `doc_splitter_request_failed`) en vez de agruparlos bajo uno solo. Sirven para elegir estrategia de retry: `pdf_slice_upload_failed` y `doc_splitter_request_failed` suelen resolverse reintentando; `pdf_slice_failed` (PDF corrupto o descarga fallida) requiere reenviar el archivo con un `file_url` nuevo.
- **Aclaración sobre `options.uploaded_file`**: no es una propiedad de creación en `add-items` (no aparece en `options.properties` del schema). Es un **flag booleano** (`true`) que solo se envía en `PUT /verification-items/{id}` como paso 3 del [flujo de carga directa](/guia-devs/crear-verificaciones/via-api/carga-directa), después de subir el archivo al `upload_url` presignado. La spec, la guía de `doc_splitter` y la nota de novedades quedaron corregidas para no listarlo como alternativa a `file_url` en creación.
- **Nuevo tipo de ítem `doc_splitter`**: divide un PDF con varios documentos y devuelve cortes (`split_documents`) con `support_id`, `page_start`, `page_end` y `support_metadata`. Se crea con `POST /verifications` o `PUT /verifications/{id}/add-items` (mismos endpoints que cualquier otro ítem). Impacto en integraciones:
  - **Referenciar un corte desde otro ítem**: nuevo `options.file_source: "item"` + `options.file_source_info: { item_id, support_id }` en `add-items`. `item_id` es el `id` del ítem `doc_splitter`; `support_id` es el `support_id` del corte. No envíes `bucket`/`key`/`support_url`: Trébol resuelve el sub-PDF internamente.
  - **Opciones**: `allowed_item_types` (array de tipos built-in y `cit_*`) o `client_item_type` (atajo para uno solo). Si omites ambos, se usa el catálogo base completo.
  - **`item_error` posibles al fallar**: `password_protected_pdf`, `unsupported_file_type`, `unknown_custom_item_type`, `misconfigured_custom_item_type`, `no_splits_returned`, `pdf_slice_failed`, `pdf_slice_upload_failed`, `doc_splitter_request_failed`.
  - **Solo acepta PDF** como archivo de entrada.
  - Copia del OpenAPI sincronizada: `add-items` ahora lista `doc_splitter` en el enum de `type` y documenta las opciones `file_source` / `file_source_info`.
## 2026-07-30

- **Dominio personalizado de onboarding**: la cuenta puede configurar su propio dominio para las ligas de onboarding desde Personalización (DNS + verificación). Con el dominio verificado, las ligas de inicio quedan `https://<dominio>/<id_slug>` y `onboarding_url` cambia de host conservando su formato. El widget embebido sigue usando el dominio por defecto. Actualizados `flows/widget.md` y la referencia openapi.

## 2026-07-16

- **Campo `classification_status` renombrado a `status` en tipos de ítem personalizados** (`/v2/custom-item-types`): el backend volvió al nombre original `status`. La spec, la guía y el reference del skill quedaron alineados. Impacto en integraciones:
  - Respuestas de `POST`, `GET`, `PATCH` y `LIST` de `/v2/custom-item-types` exponen `status` en lugar de `classification_status`.
  - `PATCH /v2/custom-item-types/{id}` acepta `status` (en lugar de `classification_status`) para archivar/reactivar un tipo (`"active"` | `"archived"`).
  - Si tu código consumía o enviaba `classification_status`, actualízalo a `status`.

## 2026-07-08

- **CRUD de tipos de ítem personalizados actualizado** (`/v2/custom-item-types`): la spec y la guía se alinearon con el backend. Cambios que afectan la integración:
  - **Campos renombrados**: `prompt_id` → `process_reference` en los procesos; el campo de estado del tipo `status` → `classification_status` (en respuestas y en el PATCH del tipo).
  - **`name` debe empezar con el prefijo `cit_`** y no puede contener espacios (usa `_`/`-`). Al crear un tipo ya **no** se envía `process_reference`: el de clasificación se deriva como `<name>_classification`.
  - **Nuevo endpoint `DELETE /v2/custom-item-types/{id}`**: elimina el tipo de forma permanente e irreversible. Para pausarlo (reversible) se sigue usando PATCH `classification_status: "archived"`.
  - **Límites de procesos**: 1 de clasificación, hasta 5 de extracción y hasta 20 de validación por tipo (409 al exceder).
  - **PATCH de proceso** ahora acepta `json_schema` (solo extracción). Eliminar el proceso de clasificación devuelve 409.
  - `friendly_name`: entre 2 y 255 caracteres.
## 2026-07-07

- **Nuevo webhook `verification.v2.document_status_updated`**: se dispara en cada transición del estado documental (`documents_status`) de una verificación, con `previous_documents_status` y `updated_at` en el payload. Permite detectar cuándo el prospecto completa su expediente (`full_upload`) y calcular tiempos de respuesta sin polling. A diferencia de los demás eventos v2, no incluye `account_name`.
- **Enum de `documents_status` corregido en el OpenAPI**: se agregó el valor `pending_external` (documentos completos con pasos externos pendientes, como formularios o UBOs), que faltaba en el spec.
- **Copia del OpenAPI sincronizada**: `reference/openapi.yaml` vuelve a coincidir con el spec canónico (incluye también los parámetros `with_citations` de coordenadas de citas).

## 2026-06-17

- **Pipeline de configuración de plantillas de dictamen**: nuevo sub-pipeline `dictamen-template-pipeline/` integrado dentro del skill de Trébol. Permite configurar plantillas Word (.docx) y PDF de clientes insertando las variables de Trébol en los lugares correctos, para que el endpoint de exportación (`GET`/`POST /v2/verifications/{id}/export/{doc-template-id}`) las autollene. Incluye 6 stages, panel de revisión con 3 personas, helpers Word/PDF y catálogo completo de variables.
- **Endpoint de exportación documentado**: contrato completo del GET y del POST con `key_people` para apoderados; aclarado que la respuesta es `{ download_url }` (no el documento directamente).
- **Catálogo de variables ampliado**: familia `shareholder_person_0_*` (accionista persona física) documentada con campos confirmados contra payload real.
- **Consultas públicas externas**: el OpenAPI ahora documenta la sección `external-lookups` en `GET /v2/verifications/{id}/{entity}` y `GET /v2/companies/{tag}/{section}` (evidencia de auditoría de las consultas a SAT, RENAPO, INE y SIGER: dependencia, sujeto, fecha, resultado y artefactos). Se agregaron los schemas `V2ExternalLookup`, `V2ExternalLookupEvidence`, `V2ExternalLookupsData` y `V2PublicResponseExternalLookups`.
- **Reporte PDF de auditoría**: `GET /verifications/{id}` ahora puede incluir `lookups_report` (`url` firmada + `generated_at`) para descargar el reporte PDF estandarizado de las consultas externas.

## 2026-05-13

- **Cobertura expandida**: agregados stubs para los otros casos de uso de Trébol — `flows/kyb-colombia.md`, `flows/kyb-eeuu.md`, `flows/hipotecas.md` y `flows/nomina.md`. Cada stub incluye items disponibles, ejemplo de payload, reglas de oro del caso y links a la doc canónica para profundizar. Antes el skill solo tenía walk-through detallado de KYB México; ahora cubre toda la matriz de casos de uso documentados.

## 2026-05-12

- **Migración a skills.sh**: el skill ahora se instala con `npx skills add gotrebol/skills` y se actualiza con `npx skills update`. Soporta múltiples editores con IA (Claude Code, Cursor, Codex, GitHub Copilot, Windsurf y otros). Antes solo funcionaba como plugin de Claude Code.
- **Regla de auto-frescura**: si pasaron más de 30 días desde la última actualización del skill, el asistente le recuerda al dev correr `npx skills update` al final de sus respuestas sobre Trébol.
- **CHANGELOG público**: este archivo. Permite al asistente responder preguntas sobre qué cambió en versiones recientes.

## Versiones anteriores

Para cambios previos a 2026-05-12, ver el historial de commits del repo público:

https://github.com/gotrebol/skills/commits/main/skills/trebol
