# Flujo: KYB Colombia vía API

## Contexto

KYB Colombia permite verificar empresas colombianas (S.A.S., S.A., Ltda., etc.), representantes legales y estructura accionaria. Trébol consulta fuentes oficiales (**RUES**, **DIAN**, **Cámara de Comercio**) y extrae información de RUT y certificados de representación legal.

## Endpoint base

```
POST https://api.gotrebol.com/verifications
```

## Items disponibles para KYB CO

### Documentos (requieren `file_url`)

| Item | Documento |
|---|---|
| `rut_co` | Registro Único Tributario (RUT) |
| `cc_co_ops` | Certificado de Cámara de Comercio (cuando el cliente lo sube) |

### Consultas públicas (no requieren archivo)

| Item | Qué consulta |
|---|---|
| `rues` | NIT en el Registro Único Empresarial |
| `public_rut_co` | NIT en la DIAN (RUT público) |
| `public_address_cc_co` | Dirección y contacto en Cámara de Comercio pública |
| `cc_co_ops` | Certificado de existencia y representación legal (Trébol lo obtiene de Cámara de Comercio) |

### Globales aplicables a CO

`person_id` (cédula de ciudadanía, pasaporte), `proof_address`, `bank_statement`, `generic`.

## Ejemplo — Consulta simple por NIT (sin documentos)

```json
{
  "country": "co",
  "tag": "uuid-cliente-1234",
  "tax_id": "900123456-7",
  "items": [
    { "type": "rues", "options": { "nit": "900123456" } },
    { "type": "public_rut_co", "options": { "nit": "900123456" } },
    { "type": "public_address_cc_co", "options": { "nit": "900123456" } },
    { "type": "cc_co_ops", "options": { "nit": "900123456" } }
  ]
}
```

## Reglas de oro para KYB CO

1. **`country: "co"`** — Colombia sí está en el enum de países soportados (`["mx", "co"]`). Úsalo literal.
2. **NIT con o sin dígito de verificación:**
   - En `tax_id` (a nivel verificación): puede incluir el dígito (`900123456-7`).
   - En `options.nit` de items: usualmente **sin** dígito (`900123456`).
3. **`cc_co_ops` es dual** — el mismo item funciona como consulta pública (pasando solo `nit`) **o** como documento cargable (pasando `file_url`). Elige según tu flujo.
4. **Combina consultas públicas + documentos en la misma verificación** — el ejemplo de "Combinación completa" en la doc canónica muestra el patrón realista de onboarding.
5. **Para extraer estructura accionaria**: usar items de tipo `ubos` con `ubos_form_schema: "co_form"` (solo aplicable en account-flows, no en verificaciones directas).

## Fuente canónica (más detalle)

- Overview, variaciones de creación y ejemplos completos: `guia-devs/uso-kyb/colombia/overview.mdx`
- Items de documentos (estructura de respuesta): `guia-devs/uso-kyb/colombia/items-documentos.mdx`
- Items de consultas públicas (RUES, DIAN, Cámara): `guia-devs/uso-kyb/colombia/items-consultas-publicas.mdx`
- Reglas de validación (`vr_trebol_*`): `guia-devs/crear-verificaciones/via-api/reglas-validacion.mdx`

Para schemas exactos y status codes, ver `reference/openapi.yaml`.
