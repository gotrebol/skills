# Flujo: Widget de onboarding embebido

## Cuándo usar el widget vs API directa

| Situación | Usa |
|---|---|
| **Tu usuario final (prospecto)** sube documentos desde tu app | Widget |
| **Tu backend** procesa documentos que ya tienes | API directa |

El widget es ideal cuando:
- Tienes una app propia y quieres onboarding embebido
- Tus prospectos no son técnicos — solo necesitan subir archivos
- Quieres una UX guiada sin construirla tú

## Diferencias clave

| | Widget | API directa |
|---|---|---|
| Quién carga el doc | Usuario final | El cliente (tú) |
| Requiere `account-flow` | Sí | No |
| Validación | Siempre | Opcional |
| Extracción | Siempre | Opcional |
| UI/UX | Trébol la provee | El cliente la construye |

## Loop completo (frontend + backend)

```
[FRONTEND]
1. Generas un identificador único por usuario (tag).
2. Embebes <trebol-widget> con flowid + client + redirecturl + tag.
3. Usuario sube documentos en el widget.
4. Widget redirige a tu redirecturl al terminar.

[TRÉBOL — asíncrono]
5. Trébol valida y extrae cada documento (puede tomar segundos a minutos).
6. Trébol envía webhooks: verification_item.v2.completed (por item)
   y verification.v2.finished (cuando todos los items terminan).

[BACKEND]
7. Tu endpoint de webhook valida la firma HMAC, encola, responde 200.
8. Worker procesa: lee resultados con
   GET /v2/companies/{etiqueta}/details
   o GET /v2/verifications/{verification-id}/people
9. Marca al usuario como "verificado" en tu DB.
```

## Instalar el widget en tu HTML

### Paso 1: Cargar el script desde el CDN

```html
<script src="https://onboarding.gotrebol.com/widget/trebol-button.js" defer></script>
```

Va dentro de `<head>` o antes del cierre de `<body>`. `defer` asegura que el script no bloquee el render del HTML.

### Paso 2: Insertar el componente

```html
<trebol-widget
  flowid="tu-id-slug"
  client="123e4567-e89b-12d3-a456-426614174000"
  redirecturl="https://tusitio.com/confirmacion?state=xyz"
  tag="etiqueta-para-verificacion">
</trebol-widget>
```

### Atributos

| Atributo | Qué es | Cómo obtenerlo |
|---|---|---|
| `flowid` | El `id_slug` del account-flow | De la respuesta de `POST /account-flows`. Ver sección "Crear el account-flow" más abajo |
| `client` | Tu `account_id` (UUID) | Desde [app.gotrebol.com](https://app.gotrebol.com), en la sección de Ajustes / Cuenta del dashboard. Si no lo encuentras, contacta a Trébol |
| `redirecturl` | A dónde se redirige al usuario al terminar | URL de tu app. Trébol redirige incluso si el flujo se interrumpe — valida en el backend con webhooks, no asumas éxito por la redirección |
| `tag` | Tu identificador para la verificación creada | Genera uno por usuario (UUID). Lo usas después para leer resultados |

### Comportamiento de `redirecturl`

La redirección ocurre cuando el usuario termina el flujo del widget (envía o cancela). El estado real de la verificación **lo confirmas con webhooks**, no con la URL de redirección.

Patrón recomendado:
1. La página de `redirecturl` muestra un mensaje genérico ("Estamos procesando tu información...")
2. Tu frontend hace polling a tu propio backend cada N segundos
3. Tu backend devuelve el estado real (que actualizaste con los webhooks)
4. Cuando el backend confirma `finished`, muestras el siguiente paso al usuario

Para el detalle exacto de cómo Trébol llama a `redirecturl` (query params, body, eventos), consulta la documentación oficial: https://docs.gotrebol.com/guia-devs/crear-verificaciones/via-widget/instalar

## Ejemplo HTML completo

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Onboarding</title>
  <script src="https://onboarding.gotrebol.com/widget/trebol-button.js" defer></script>
</head>
<body>
  <h1>Bienvenido al onboarding</h1>
  <p>Por favor carga los documentos necesarios.</p>

  <trebol-widget
    flowid="onboarding-mx-estandar"
    client="123e4567-e89b-12d3-a456-426614174000"
    redirecturl="https://tusitio.com/confirmacion"
    tag="registro-2024-001">
  </trebol-widget>
</body>
</html>
```

## Crear el account-flow (vía API, antes de embeber)

Endpoint: `POST /account-flows`

Un flow tiene dos partes:

- **`record_validation_schema`** — qué documentos pedir y con qué reglas. Cada requerimiento es un slot: `doc_1`, `doc_2`, etc.
- **`flow_items`** — items adicionales no-documento: UBOs, formularios personalizados, consultas SAT.

### Ejemplo mínimo

```json
{
  "friendly_name": "Onboarding MX Estándar",
  "id_slug": "onboarding-mx-estandar",
  "country": "mx",
  "record_validation_schema": {
    "version": 2,
    "requirements": {
      "doc_1": {
        "allowed_item_types": ["csf_mx"],
        "ui_options": {
          "label": "Constancia de Situación Fiscal",
          "description": "CSF con menos de 60 días de antigüedad"
        },
        "validation_options": {
          "on_invalid_type_error": "invalidate",
          "ruleset": [
            {
              "id": "vr_trebol_antiguedad",
              "params": { "days": 60 },
              "error_message": "La CSF debe tener menos de 60 días"
            }
          ]
        }
      },
      "doc_2": {
        "allowed_item_types": ["ac_mx"],
        "ui_options": {
          "label": "Acta Constitutiva",
          "description": "Documento que certifica la constitución de la empresa"
        },
        "validation_options": { "on_invalid_type_error": "invalidate" }
      }
    }
  },
  "flow_items": {
    "items": [
      { "type": "public_sat_signatures", "options": { "type": "business" } },
      { "type": "public_sat_signatures", "options": { "type": "representatives" } },
      {
        "type": "ubos",
        "options": {
          "ubos_form_schema": "mx_form",
          "ubos_threshold": 25,
          "is_optional": false
        }
      }
    ]
  }
}
```

⚠️ Importante sobre `country` en account-flows:
- En **`POST /account-flows`** (schema `AccountFlowCreate`) el campo `country` no está declarado en la spec — se omite del body de creación.
- En **`PUT` y `GET` de account-flows** sí aparece, con enum `["mx", "co"]`.
- Si necesitas crear un flow para EEUU, consulta con Trébol antes de inventar un valor de país.

## Items del flow

Los más usados:

| Item | Para qué | País |
|---|---|---|
| `ubos` | Formulario de declaración de beneficiarios reales | Cualquiera |
| `forms` | Formulario personalizado (requiere `schema_id` previamente creado vía `POST /v2/form-schemas`) | Cualquiera |
| `public_sat_signatures` | Consulta automática de FIEL/sellos en SAT | MX |
| `siger` | Sistema Integral de Gestión Registral | MX |
| `curp_item` | Validación de CURP | MX |
| `rues` | Registro Único Empresarial y Social | CO |

Para la lista completa de items soportados (incluyendo aml_validation, signatory_validation, items por país), consulta `reference/openapi.yaml` o https://docs.gotrebol.com/guia-devs/referencia/tipos-item.

## Personalización (branding)

Configura colores, logo, textos y políticas desde [app.gotrebol.com](https://app.gotrebol.com) → Personalización. No requiere código.

## Estados del expediente

A medida que el usuario carga documentos, el expediente pasa por estados. Para monitorear, usa webhooks (`verification.v2.created`, `verification_item.v2.completed`, `verification.v2.finished`) o consulta:

```
GET /v2/verifications/{verification-id}/details
```

## Errores comunes al instalar el widget

| Síntoma | Causa probable |
|---|---|
| El botón no aparece | Falta `<script>` o cargó después del componente sin `defer` |
| `flowid not found` | `id_slug` mal escrito o no existe en tu cuenta |
| `unauthorized client` | `account_id` incorrecto |
| Redirect no funciona | `redirecturl` debe ser absoluta y `https://` en producción |
