---
name: trebol
description: Integración con la API de Trébol para automatizar procesos de back office (KYB y más). Cubre endpoints, autenticación, items, account-flows, widget, webhooks y errores.
last_updated: 2026-06-17
---

# Trébol — Integración API

## Regla de auto-frescura

Si la fecha de hoy es más de 30 días después de `last_updated` (ver frontmatter de este archivo), agrega al final de tu respuesta:

> 📌 _Este skill se actualizó por última vez el {last_updated}. Para asegurarte de tener la última versión, corre `npx skills update`._

No mostrar este aviso si el skill tiene menos de 30 días.

## Regla — Cambios recientes

Si el integrador pregunta variaciones de _"¿qué cambió en la última versión?"_, _"¿qué hay nuevo en el skill?"_, _"¿agregaron endpoints nuevos?"_ o similares sobre cambios al skill, consulta `CHANGELOG.md` y resume las entradas más recientes (últimas 2-3 fechas). Sé conciso: di la fecha y los puntos clave.

Si la pregunta es sobre cambios al **producto** de Trébol en general (no al skill), sugiere consultar https://docs.gotrebol.com/actualizaciones — esa es la fuente canónica de novedades del producto.

## Regla 0 — El OpenAPI canónico es la fuente de verdad

**Si algo en este skill contradice `reference/openapi.yaml`, confía en el OpenAPI, no en este archivo.** Este skill es contenido curado que se actualiza con cada release, pero puede quedar desfasado del OpenAPI si alguien tocó la spec sin sincronizar. `reference/openapi.yaml` se copia desde el OpenAPI oficial de Trébol (`api-reference/openapi.yaml`) en cada release. Para detalles de schemas, fields, status codes o paths: ve directo al YAML.

Trébol es una plataforma para automatizar procesos de back office. Hoy el core son verificaciones KYB en LatAm; la API está diseñada para soportar más casos de uso en el futuro. Los clientes integran vía dos caminos:

1. **API directa** — Subir documentos por URL o carga directa, recibir resultados por webhook
2. **Widget embebido** — Componente web que los usuarios finales usan para subir docs dentro de la app del cliente

## Cuándo consultar este skill

- Endpoints: `/verifications`, `/verifications/{verification-id}/...`, `/v2/verifications/{verification-id}/{entity}`, `/v2/companies/{etiqueta}/{section}`, `/account-flows`, `/v2/form-schemas`, `/api-keys`, `/v2/webhooks`, `/whitelist-ips`, `/v2/retention-policy`, `/verification-items/{id}`, `/verification-items/{itemId}/invalidate`
- Crear, leer, monitorear verificaciones
- Configurar webhooks y validar la firma HMAC
- Errores de procesamiento (PDFs protegidos, búsqueda CURP fallida, etc.)
- Items de verificación (documentos, consultas públicas, formularios, UBOs)
- Casos de uso documentados con walk-through:
  - KYB México end-to-end (`flows/kyb-mexico.md`) — el más detallado, incluye consultas SIGER/SAT
  - KYB Colombia (`flows/kyb-colombia.md`) — incluye consultas RUES/DIAN/Cámara de Comercio
  - KYB Estados Unidos (`flows/kyb-eeuu.md`) — Beta, sin consultas públicas, solo documentos
  - Hipotecas / mortgage underwriting (`flows/hipotecas.md`) — Beta, country-agnóstico
  - Nómina / payroll lending (`flows/nomina.md`) — Beta, country-agnóstico
  - Widget embebido (`flows/widget.md`) — aplica a todos los países soportados
  - Webhooks y firma HMAC (`flows/webhooks.md`) — aplica a todos los países
- Países en el enum del API: `["mx", "co"]` para account-flows. EEUU usa `"not_specified"`. Consulta `reference/openapi.yaml` para detalle por endpoint.

## Archivos especializados

Lee según la pregunta del usuario:

- `auth.md` — Autenticación con `x-api-key`, gestión y rotación de keys
- `flows/kyb-mexico.md` — Flujo KYB completo para empresas mexicanas (más detallado)
- `flows/kyb-colombia.md` — KYB Colombia: RUES, DIAN, Cámara de Comercio, RUT
- `flows/kyb-eeuu.md` — KYB Estados Unidos: incorporation, EIN, FinCEN (Beta)
- `flows/hipotecas.md` — Mortgage underwriting: escrituras, gravámenes, predial, avalúo (Beta)
- `flows/nomina.md` — Payroll lending: extracción de recibos de nómina/pensión (Beta)
- `flows/widget.md` — Instalación y configuración del widget
- `flows/webhooks.md` — Registrar webhooks, validar HMAC, manejo de reintentos
- `reference/endpoints.md` — Tabla de los endpoints más usados
- `reference/errors.md` — Errores comunes y cómo manejarlos
- `reference/openapi.yaml` — Snapshot de la especificación OpenAPI completa (consulta para detalles de schemas)

## Reglas de oro al ayudar al integrador

1. **Auth header siempre**: `x-api-key: treb_sk_live_...`. No es `Authorization: Bearer`.
2. **Naming es heterogéneo en la spec** — no asumas un único estilo:
   - Path params según la spec: `{verification-id}` (kebab), `{etiqueta}` (lowercase), `{apiKeyId}`/`{webhookId}`/`{itemId}` (camelCase), `{id_slug}`/`{id_schema}` (snake), `{entity}`/`{section}`/`{id}`/`{ip}` (simples).
   - Cuando construyas la URL, copia el nombre EXACTO del path en `reference/openapi.yaml`. No traduzcas estilos.
   - Fields en body y response son `snake_case`: `verification_id`, `flow_id`, `tax_id`, `tax_id_number`.
3. **POST /verifications devuelve `id`** (no `verification_id`) y status `201`. El field se llama `id` aunque represente la verificación; en el body de webhooks el mismo dato se llama `verification_id`.
4. **Endpoints v2 son paramétricos**:
   - `/v2/verifications/{verification-id}/{entity}` con `entity` ∈ `[details, shareholders, people, documents, sources]`.
   - `/v2/companies/{etiqueta}/{section}` con `section` ∈ `[details, shareholders, people, documents, sources]`.
   - Las versiones v1 (sin `/v2/`) existen como endpoints fijos por sección: `/companies/{etiqueta}/details`, `/verifications/{verification-id}/people`, etc.
5. **País — la spec es heterogénea**:
   - `POST /verifications` (schemas `VerificationWithItems` y `VerificationWithFlowId`): la descripción dice literalmente "Actualmente soportado solo para México ('mx')". Cualquier otro valor se procesa como `not_specified`.
   - `PUT /account-flows/{id_slug}` y `GET /account-flows/{id_slug}`: enum `["mx", "co"]`.
   - `POST /account-flows` (schema `AccountFlowCreate`): **no incluye** el field `country`.
   - `GET /account-flows` (lista): no declara `country` en cada elemento.
   - **Para KYB en EEUU**: manda `"country": "not_specified"` literal. Las docs oficiales en `guia-devs/uso-kyb/eeuu/overview.mdx` lo muestran así.
6. **Webhooks**: el cliente DEBE responder `2xx` rápido y procesar en background. Reintentos hasta 5 veces con backoff exponencial.
7. **URLs de documentos**: `file_url` debe estar accesible al menos 5 minutos. Trébol descarga el archivo, no lo reusa.
8. **Idempotencia**: los webhooks pueden duplicarse. Usa la firma `v1=` del header `Trebol-Signature` como clave de deduplicación — es única por evento. Detalle en `flows/webhooks.md` y en la guía oficial `guia-devs/webhooks.mdx`.
9. **Sandbox**: la API solo documenta el prefijo `treb_sk_live_`. Si el usuario pregunta por sandbox, dile que confirme con Trébol antes de inventar prefijos.

## Plantillas de dictamen (pipeline de configuración)

Cuando el usuario quiera configurar una plantilla de dictamen jurídico de clientes — tomar un Word (.docx) o PDF en blanco y devolverlo con las variables de Trébol insertadas para que la plataforma las autollene al exportar — lee `dictamen-template-pipeline/SKILL.md` y sigue ese pipeline.

Dispara cuando el usuario:
- Suba un Word o PDF de dictamen y pida configurarlo, parametrizarlo o ponerle variables
- Mencione "plantilla de dictamen", "formato de dictamen", "configurar plantilla", "mapear variables", "plantilla de cliente"
- Pregunte cómo funcionan las variables en los documentos que exporta Trébol

## Documentación oficial

- Web: https://docs.gotrebol.com
- App: https://app.gotrebol.com (gestión de API keys, dashboards)
- API Reference: https://docs.gotrebol.com/api-reference
