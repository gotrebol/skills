# Endpoints más usados

URL base: `https://api.gotrebol.com`

Todos los endpoints requieren header `x-api-key: tu_api_key`.

## Convención de naming (importante: heterogénea)

La spec NO usa un único estilo de naming para path params. Cada endpoint declara el suyo. Cuando construyas un path, copia el nombre exacto del spec:

| Tipo de param | Ejemplos en la spec |
|---|---|
| kebab-case | `{verification-id}`, `{doc-template-id}` |
| camelCase | `{apiKeyId}`, `{webhookId}`, `{itemId}` |
| snake_case | `{id_slug}`, `{id_schema}` |
| lowercase / palabra simple | `{etiqueta}`, `{entity}`, `{section}`, `{id}`, `{ip}` |

Los **fields del body y response** sí son consistentes: `snake_case` (`verification_id`, `flow_id`, `tax_id`, `tax_id_number`).

⚠️ Esto significa que un mismo concepto puede aparecer con grafías distintas: lo recibes como `verification_id` en el body de un webhook y lo pasas como `{verification-id}` al construir el path.

## Top endpoints

| Método | Path | Para qué |
|---|---|---|
| `POST` | `/verifications` | **Crear verificación** (KYB, hipotecas, nómina, etc.). Devuelve `id` (no `verification_id`) y status `201` |
| `GET` | `/verifications` | Listar verificaciones de la cuenta |
| `GET` | `/verifications/{verification-id}` | Detalles de una verificación específica |
| `GET` | `/v2/verifications/{verification-id}/{entity}` | Sección específica de la verificación. `entity` ∈ `[details, shareholders, people, documents, sources]` |
| `GET` | `/v2/companies/{etiqueta}/{section}` | Sección consolidada de la empresa por tu tag. `section` ∈ `[details, shareholders, people, documents, sources]` |
| `GET` | `/companies/{etiqueta}/details` | Versión v1 fija de "details" (legacy, sigue funcionando) |
| `POST` | `/account-flows` | Crear flow de widget (define documentos, reglas, items) |
| `POST` | `/v2/webhooks` | Registrar webhook |
| `GET` | `/v2/webhooks` | Listar webhooks |
| `POST` | `/api-keys` | Crear API key (programática) |
| `GET` | `/api-keys` | Listar API keys |

ℹ️ Los endpoints v2 (`/v2/verifications/{verification-id}/{entity}` y `/v2/companies/{etiqueta}/{section}`) son **paramétricos**: tú pasas la sección como path segment. Los v1 sin `/v2/` están como endpoints fijos por sección.

## Patrones comunes

### Crear, esperar webhook, leer

```
POST /verifications                                            ← request
↓ Trébol responde con id (síncrono, 201)
↓ Trébol procesa (asíncrono, segundos a minutos)
← webhook verification_item.v2.completed (por cada item)
← webhook verification.v2.finished
GET /v2/companies/{etiqueta}/details                           ← leer info final
```

### Extraer accionistas/apoderados después de procesar acta

```
POST /verifications con item ac_mx (acta constitutiva)
↓
← webhook verification_item.v2.completed (acta procesada)
← webhook verification_people.curp_search_completed (CURP de cada accionista)
GET /v2/verifications/{verification-id}/people                 ← leer personas extraídas
```

### Cuándo usar `/v2/companies/{etiqueta}/...` vs `/v2/verifications/{verification-id}/...`

- **Companies (por `etiqueta`)** — vista consolidada de la **empresa** asociada a tu tag. Buena para mostrar al usuario final.
- **Verifications (por `verification-id`)** — vista técnica de una **verificación específica**. Útil para correlacionar con tu tracking interno.

## Códigos de respuesta

| HTTP | Significado |
|---|---|
| `200` | OK |
| `201` | Recurso creado |
| `400` | Body mal formado o validación fallida |
| `401` | API key inválida o ausente |
| `403` | API key no tiene permisos para esa acción |
| `404` | Recurso no existe |
| `409` | Conflicto (ej. crear con un `id` duplicado) |
| `500` | Error interno de Trébol |

## Formato de respuesta de error

Todos los errores siguen esta estructura:

```json
{
  "success": false,
  "message": "Descripción accionable",
  "code": "VALIDATION_ERROR",
  "timestamp": "2025-01-01T12:34:56.000Z"
}
```

Programa tus clientes en función de `code` y HTTP status, no del texto de `message`.

## Documentación detallada

Para schemas completos, parámetros opcionales y todos los endpoints:
- API Reference oficial: https://docs.gotrebol.com/api-reference
- OpenAPI spec local: `reference/openapi.yaml` (en este mismo skill, snapshot al momento de publicar el paquete)
