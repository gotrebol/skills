# Flujo: Nómina (Payroll Lending) vía API

## Contexto

El caso de uso de **nómina** permite verificar ingresos laborales (recibos de nómina), pensiones, historial de pagos, descuentos de ley (salud, pensión, retención), y créditos/embargos registrados en el comprobante. Trébol extrae de forma estructurada los campos clave del recibo: ingresos recurrentes y no recurrentes, descuentos, créditos, embargos e ingreso neto.

> **Estado: Beta.** El item `payroll_receipt` está marcado como **Beta** en la API hoy.

## Endpoint base

```
POST https://api.gotrebol.com/verifications
```

## Items disponibles

### Documento específico (requiere `file_url`)

| Item | Documento |
|---|---|
| `payroll_receipt` | Recibo de pago de nómina o pensión (Beta) |

### Globales aplicables

`person_id` (identificación del empleado/pensionado), `proof_address`, `bank_statement`, `generic`.

## Ejemplo — Clasificación automática + extracción

```json
{
  "country": "co",
  "tag": "uuid-empleado-1234",
  "items": [
    {
      "type": "generic",
      "options": {
        "file_url": "https://www.ejemplo.com/recibo-nomina.pdf",
        "client_item_type": "payroll_receipt"
      }
    },
    {
      "type": "generic",
      "options": {
        "file_url": "https://www.ejemplo.com/cedula.pdf"
      }
    }
  ]
}
```

## Reglas de oro para Nómina

1. **Item country-agnóstico** — `payroll_receipt` aplica a cualquier país. La estructura de respuesta incluye campos de identificación comunes a varios países (CC Colombia, RFC México, pasaporte, etc.).
2. **`country` configurable** — usa `"mx"` o `"co"` según el origen del recibo. Para otros países, `"not_specified"`.
3. **Item en Beta** — puede cambiar campos sin previo aviso. Validar contra el OpenAPI antes de cada release de tu integración.
4. **Combina con `person_id`** — el flujo típico para payroll lending incluye verificar la identidad del empleado/pensionado además de extraer el recibo.
5. **Campos extraídos típicos** — ingreso bruto, ingreso neto, descuentos de ley (salud, pensión, retención), descuentos voluntarios (créditos, embargos), periodo del pago. Ver detalle exacto en la doc canónica.
6. **Recibos digitales vs físicos** — Trébol acepta PDFs digitales o escaneos. Calidad del escaneo afecta la confianza de extracción.

## Fuente canónica (más detalle)

- Overview, variaciones de creación y ejemplos completos: `guia-devs/uso-nomina/overview.mdx`
- Items de nómina (estructura de respuesta detallada): `guia-devs/uso-nomina/items.mdx`
- Reglas de validación (`vr_trebol_*`): `guia-devs/crear-verificaciones/via-api/reglas-validacion.mdx`

Para schemas exactos y status codes, ver `reference/openapi.yaml`.
