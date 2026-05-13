# Flujo: KYB Estados Unidos vía API

## Contexto

KYB EEUU permite verificar corporaciones registradas en los estados de EEUU, estructura corporativa (directores, accionistas, incorporators), identificación fiscal federal (EIN) y registros en FinCEN. Trébol automatiza la extracción de documentos oficiales de constitución y registro.

> **Estado: Beta.** Los items de KYB EEUU están marcados como **Beta** en la API hoy. Sin consultas públicas automatizadas (a diferencia de MX y CO); todo se hace por documento.

## Endpoint base

```
POST https://api.gotrebol.com/verifications
```

## Items disponibles para KYB EEUU

### Documentos (requieren `file_url`)

| Item | Documento |
|---|---|
| `certificate_of_incorporation_extractor` | Certificado de constitución (Beta) |
| `certificate_of_incumbency_extractor` | Certificado de incumbencia (Beta) |
| `irs_ein_assignment_letter_extractor` | Carta de asignación de EIN del IRS (Beta) |
| `fincen_msb_registration_extractor` | Registro de MSB (Money Services Business) en FinCEN (Beta) |

### Globales aplicables a EEUU

`person_id` (pasaporte, etc.), `proof_address`, `bank_statement`, `generic`.

## Ejemplo — Clasificación automática + extracción

```json
{
  "country": "not_specified",
  "tag": "uuid-cliente-us-1234",
  "items": [
    {
      "type": "generic",
      "options": {
        "file_url": "https://www.ejemplo.com/certificate-of-incorporation.pdf",
        "client_item_type": "certificate_of_incorporation_extractor"
      }
    },
    {
      "type": "generic",
      "options": {
        "file_url": "https://www.ejemplo.com/irs-ein-letter.pdf"
      }
    }
  ]
}
```

## Reglas de oro para KYB EEUU

1. **`country: "not_specified"`** literal. EEUU **no** está en el enum de países soportados (`["mx", "co"]`); cualquier otro valor incluyendo `"us"` o `"usa"` se procesa como `not_specified` (modo genérico, sin validaciones de formato locales).
2. **Items en Beta** — los 4 items específicos de EEUU están en Beta. Pueden cambiar campos en la respuesta sin previo aviso. Para producción, validar contra `reference/openapi.yaml` antes de cada release.
3. **Sin consultas públicas automatizadas** — a diferencia de MX (SIGER, SAT) y CO (RUES, DIAN), en EEUU todo se procesa por documento. El cliente debe subir los archivos.
4. **EIN format** — la carta del IRS contiene el EIN en formato `XX-XXXXXXX`. Trébol lo extrae tal cual, sin reformatear.
5. **FinCEN MSB** — específico para fintechs y empresas reguladas; opcional para la mayoría de empresas de cliente final.

## Fuente canónica (más detalle)

- Overview, variaciones de creación y ejemplos completos: `guia-devs/uso-kyb/eeuu/overview.mdx`
- Items de documentos (estructura de respuesta): `guia-devs/uso-kyb/eeuu/items-documentos.mdx`
- Reglas de validación (`vr_trebol_*`): `guia-devs/crear-verificaciones/via-api/reglas-validacion.mdx`

Para schemas exactos y status codes, ver `reference/openapi.yaml`.
