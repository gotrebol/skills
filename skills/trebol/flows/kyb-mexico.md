# Flujo: KYB México vía API

## Contexto

KYB México permite verificar:
- **Personas morales** (S.A. de C.V., S. de R.L., S.A.P.I.)
- **Personas físicas con actividad empresarial**
- **Representantes legales y apoderados**

Trébol automatiza extracción de actas, poderes, constancias fiscales y consultas a SIGER y SAT.

## Endpoint base

```
POST https://api.gotrebol.com/verifications
```

## Las 3 formas de procesar documentos

| Forma | `type` del item | Trébol hace... |
|---|---|---|
| **Clasificación automática** | `generic` | Detecta el tipo y extrae la info |
| **Validación** | `doc_validation` | Verifica que sea del tipo esperado y cumpla reglas. **No extrae** |
| **Extracción directa** | `ac_mx`, `csf_mx`, etc. | Extrae asumiendo el tipo. **No valida** |

**Recomendación general**: empezar con `generic` (clasificación automática) y dejar que Trébol determine el tipo.

## Items disponibles para KYB MX

### Documentos (requieren `file_url`)

Estos están en el enum `AllowedItemType` del openapi (usado por `record_validation_schema.requirements[].allowed_item_types` en account-flows). En `POST /verifications` el field `items[].type` es string libre, así que también valen ahí.

| Item | Documento |
|---|---|
| `ac_mx` | Acta constitutiva |
| `aa_mx` | Acta de asamblea |
| `pw_mx` | Poder notarial |
| `fme_mx` | Folio mercantil electrónico |
| `csf_mx` | Constancia de situación fiscal |
| `designacion_responsable_cumplimiento_extractor` | Designación responsable cumplimiento |
| `tax_payment_compliance` | Opinión de cumplimiento |

### Consultas públicas (no requieren archivo)

ℹ️ Estos no aparecen en el enum `AllowedItemType` (que es solo para items-documento). El field `items[].type` en `POST /verifications` es string libre, así que estos valores son válidos. La fuente canónica es la página de docs `guia-devs/uso-kyb/mexico/items-consultas-publicas`.

| Item | Qué consulta |
|---|---|
| `siger` | Sistema Integral de Gestión Registral |
| `siger_shareholders` | Empresas relacionadas por accionistas |
| `public_sat_signatures` | Firmas FIEL y sellos digitales en SAT |
| `curp_item` | Validación de CURP |

### Globales aplicables a MX

`person_id` (INE/pasaporte/residencia), `proof_address`, `bank_statement`, `generic`.

## Atributos principales del payload

| Atributo | Tipo | Requerido | Notas |
|---|---|---|---|
| `country` | string | sí | La spec dice "Actualmente soportado solo para México (`mx`)". Cualquier otro valor se procesa como `not_specified`. Para EEUU envía `"not_specified"` literal |
| `tag` | string | sí | Tu identificador interno (UUID, ID de cliente). Lo usas después para leer resultados |
| `tax_id` | string | condicional | RFC. **Siempre** requerido cuando usas `flow_id` |
| `friendly_name` | string | no | Nombre legible de la empresa |
| `email` | string | no | Email del solicitante |
| `items` | array | sí (sin flow) | Lista de items a procesar |
| `flow_id` | string | sí (con flow) | ID de un account-flow predefinido. Mutuamente excluyente con `items` |
| `metadata` | object | no | Pares clave-valor que Trébol no procesa |
| `key_people` | array | no | Apoderados a quienes extraer poderes legales |

### Reglas de `tax_id`

- Con **`flow_id`**: el `tax_id` raíz es obligatorio siempre.
- Con **`items`** (sin flow): el `tax_id` raíz es opcional. Excepción: si usas `public_sat_signatures` con `type: "business"`, debes proveer el RFC. Va en `tax_id` raíz o en `options.tax_id_number` del item SAT.

⚠️ **`file_url` debe estar disponible al menos 5 minutos.** Trébol descarga el archivo, no lo reusa.

## Ejemplo 1 — Consulta simple (solo RFC, sin documentos)

```json
{
  "country": "mx",
  "tag": "uuid-cliente-1234",
  "tax_id": "CMT2404303V2",
  "items": [
    { "type": "siger" },
    {
      "type": "public_sat_signatures",
      "options": { "type": "business", "tax_id_number": "CMT2404303V2" }
    },
    {
      "type": "public_sat_signatures",
      "options": { "type": "representatives" }
    }
  ]
}
```

Trébol consulta SIGER, FIEL de la empresa y FIEL del representante legal.

### Respuesta de POST /verifications

Status `201`. La respuesta es síncrona. El field se llama `id` (no `verification_id`) aunque represente el ID de la verificación. En el body de webhooks el mismo dato se llama `verification_id`.

```json
{
  "id": "c8dc41fc-c477-404e-aff7-b9074f86d6d1",
  "status": "pending",
  "account_id": "99999999-9999-9999-9999-999999999999",
  "created_at": "2025-04-28T20:10:06.840Z",
  "updated_at": "2025-04-28T20:10:06.840Z",
  "documents_status": "pending_upload",
  "details_url": "https://app.gotrebol.com/verifications/c8dc41fc-c477-404e-aff7-b9074f86d6d1",
  "items": [
    { "id": 25440, "item_status": "pending", "item_type": "siger", "item_internal_status": "pending_validation" }
  ]
}
```

### Smoke test ejecutable (curl end-to-end)

```bash
curl -X POST "https://api.gotrebol.com/verifications" \
  -H "x-api-key: $TREBOL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "country": "mx",
    "tag": "smoke-test-001",
    "tax_id": "CMT2404303V2",
    "items": [
      { "type": "siger" },
      {
        "type": "public_sat_signatures",
        "options": { "type": "business", "tax_id_number": "CMT2404303V2" }
      }
    ]
  }'
```

Si todo está bien, recibes status `201` con un `id` UUID. Apunta a ese ID para cruzar con los webhooks que llegarán después.

## Ejemplo 2 — Con documentos: clasificación automática

```json
{
  "country": "mx",
  "tag": "uuid-cliente-5678",
  "friendly_name": "ACME S.A. de C.V.",
  "items": [
    {
      "type": "generic",
      "options": {
        "file_url": "https://www.ejemplo.com/acta-constitutiva.pdf",
        "client_item_type": "ac_mx"
      }
    },
    {
      "type": "generic",
      "options": {
        "file_url": "https://www.ejemplo.com/ine-rep-legal.pdf",
        "client_item_type": "person_id"
      }
    }
  ],
  "key_people": [
    { "names": "Juan Pérez", "scope": ["powers"] }
  ]
}
```

`client_item_type` es **opcional** — es un hint que ayuda a la clasificación. Si lo omites, Trébol detecta el tipo solo.

## Ejemplo 3a — Solo validación (sin extraer info)

```json
{
  "country": "mx",
  "tag": "uuid-cliente-validacion",
  "items": [
    {
      "type": "doc_validation",
      "options": {
        "file_url": "https://www.ejemplo.com/acta.pdf",
        "client_item_type": "ac_mx",
        "ruleset": [
          {
            "id": "vr_trebol_entidad",
            "error_message": "El acta debe mencionar a ACME S.A. de C.V.",
            "params": { "entity_name": "ACME S.A. de C.V." }
          }
        ]
      }
    }
  ]
}
```

`client_item_type` es **obligatorio** en `doc_validation`. `ruleset` es opcional.

## Ejemplo 3b — Solo extracción directa

```json
{
  "country": "mx",
  "tag": "uuid-cliente-extraccion",
  "items": [
    {
      "type": "ac_mx",
      "options": { "file_url": "https://www.ejemplo.com/acta.pdf" }
    }
  ]
}
```

## Ejemplo 4 — SIGER en profundidad

`siger_data_extraction: true` analiza los actos registrados en SIGER y crea automáticamente items `ac_mx`, `aa_mx`, `fme_mx` para los actos relevantes.

`search_related_companies_siger: true` identifica empresas en las que participan los accionistas.

```json
{
  "country": "mx",
  "tag": "uuid-cliente-siger",
  "tax_id": "ABC123456789",
  "items": [
    {
      "type": "siger",
      "options": {
        "siger_data_extraction": true,
        "search_related_companies_siger": true
      }
    },
    {
      "type": "public_sat_signatures",
      "options": { "type": "business", "tax_id_number": "ABC123456789" }
    }
  ]
}
```

⚠️ **`siger_data_extraction: true` no funciona en verificaciones que también incluyen documentos cargados.** Si necesitas ambas cosas, sepáralas en dos verificaciones.

Si no tienes `tax_id`, declara el `legal_name` directamente:
```json
{
  "type": "siger",
  "options": {
    "siger_data_extraction": true,
    "legal_name": "EMPRESA EJEMPLO, S.A. DE C.V."
  }
}
```

## `key_people` — apoderados y poderes

Para extraer poderes legales de personas específicas:

```json
"key_people": [
  { "names": "Juan Pérez", "scope": ["powers"] },
  { "names": "María López", "scope": ["powers", "doc_validation"] }
]
```

| Scope | Qué hace |
|---|---|
| `powers` | Extrae facultades legales del poder notarial |
| `doc_validation` | Limita la verificación a validar documentos de la persona |

## Carga directa (cuando no tienes `file_url` accesible)

Si tu archivo está en almacenamiento privado (S3 con TTL corto, generación dinámica) y no puedes pasar un `file_url` accesible 5 minutos, usa carga directa: Trébol te entrega un `upload_url` por item, tú subes el archivo directamente, Trébol procesa.

Detalle del flujo en la documentación oficial: https://docs.gotrebol.com/guia-devs/crear-verificaciones/via-api/carga-directa

## Flujo completo de integración

```
1. Cliente crea verificación → POST /verifications
2. Trébol responde con `id` (síncrono — campo se llama literalmente `id` en la respuesta, no `verification_id`)
3. Trébol procesa items en background (asíncrono)
4. Trébol envía webhook verification_item.v2.completed por cada item
5. Trébol envía webhook verification.v2.finished cuando todos los items terminan
6. Cliente lee resultados:
   - GET /v2/companies/{etiqueta}/details para vista consolidada de la empresa
   - GET /v2/verifications/{verification-id}/people para personas extraídas
```

⚠️ Path params en **kebab** (`{verification-id}`, `{etiqueta}`), pero los fields del body y response son **snake_case** (`verification_id`, `account_id`).

### Respuesta de GET /v2/companies/{etiqueta}/details (recortada)

```json
{
  "tag": "uuid-cliente-1234",
  "verification_id": "5853393e-8cf7-4dc7-afd9-a92df69fff2b",
  "status": "finished",
  "company": {
    "legal_name": "ACME S.A. DE C.V.",
    "tax_id": "ABC123456789",
    "address": "Av. Reforma 100, CDMX"
  },
  "items": [
    { "type": "siger", "status": "completed" },
    { "type": "public_sat_signatures", "status": "completed" }
  ]
}
```

### Respuesta de GET /v2/verifications/{verification-id}/people (recortada)

```json
{
  "full_list": [
    {
      "people_id": 3921,
      "names": "JUAN PÉREZ",
      "roles": [{ "type": "representative" }],
      "external_identities": {
        "curp": {
          "success": true,
          "applicant_data": {
            "curp": "PEPJ800101HDFRRN09",
            "names": "JUAN",
            "first_surname": "PÉREZ"
          }
        }
      }
    }
  ]
}
```

## Ejemplo en código (Node.js)

```javascript
const { trebol } = require('./trebol-client');

async function crearVerificacionKYB(empresaId, rfc, friendlyName) {
  return await trebol('POST', '/verifications', {
    country: 'mx',
    tag: `kyb-${empresaId}`,
    tax_id: rfc,
    friendly_name: friendlyName,
    items: [
      { type: 'siger' },
      {
        type: 'public_sat_signatures',
        options: { type: 'business', tax_id_number: rfc }
      },
      {
        type: 'public_sat_signatures',
        options: { type: 'representatives' }
      }
    ]
  });
}
```
