# Flujo: Hipotecas (Mortgage Underwriting) vía API

## Contexto

El caso de uso de **hipotecas** permite verificar titularidad de inmuebles (escrituras), gravámenes (hipotecas, embargos), pago de impuesto predial y valor de inmuebles (avalúos). Trébol extrae información de documentos oficiales de propiedad.

> **Estado: Beta.** Los 4 items específicos de hipotecas están marcados como **Beta** en la API hoy.

## Endpoint base

```
POST https://api.gotrebol.com/verifications
```

## Items disponibles

### Documentos específicos (requieren `file_url`)

| Item | Documento |
|---|---|
| `property_deed` | Escritura de compraventa de inmuebles (Beta) |
| `lien_certificate` | Certificado de gravámenes (Beta) |
| `property_tax_receipt` | Recibo de pago de impuesto predial (Beta) |
| `property_appraisal` | Avalúo de inmueble (Beta) |

### Globales aplicables

`person_id` (identificación del propietario), `proof_address`, `bank_statement`, `generic`.

## Ejemplo — Clasificación automática + extracción

```json
{
  "country": "mx",
  "tag": "uuid-inmueble-1234",
  "items": [
    {
      "type": "generic",
      "options": {
        "file_url": "https://www.ejemplo.com/escritura.pdf",
        "client_item_type": "property_deed"
      }
    },
    {
      "type": "generic",
      "options": {
        "file_url": "https://www.ejemplo.com/certificado-gravamenes.pdf"
      }
    }
  ]
}
```

## Reglas de oro para Hipotecas

1. **Items country-agnósticos pero respuesta con bias regional** — los 4 items de hipotecas aplican a cualquier país, pero el formato de respuesta incluye campos específicos de México (CURP, RFC) además de campos genéricos. Confirma en `reference/openapi.yaml` los fields esperados según tu país.
2. **`country` configurable** — a diferencia de KYB EEUU, aquí sí puedes usar `"country": "mx"` o `"co"`. Si no aplica, `"not_specified"`.
3. **Items en Beta** — pueden cambiar campos sin previo aviso. Validar contra el OpenAPI antes de cada release de tu integración.
4. **Combina con `person_id` para el titular** — el flujo típico incluye verificar la identidad del propietario además de los documentos del inmueble.
5. **`property_appraisal` requiere documento de avalúo oficial** — no funciona con valoraciones informales o de portales inmobiliarios.

## Fuente canónica (más detalle)

- Overview, variaciones de creación y ejemplos completos: `guia-devs/uso-hipotecas/overview.mdx`
- Items de hipotecas (estructura de respuesta detallada): `guia-devs/uso-hipotecas/items.mdx`
- Reglas de validación (`vr_trebol_*`): `guia-devs/crear-verificaciones/via-api/reglas-validacion.mdx`

Para schemas exactos y status codes, ver `reference/openapi.yaml`.
