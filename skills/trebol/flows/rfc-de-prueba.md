# RFC de prueba — Verificaciones KYB México con respuesta fija

Fuente canónica: `guia-devs/pruebas/rfc-de-prueba.mdx` (https://docs.gotrebol.com/guia-devs/pruebas/rfc-de-prueba). Si algo aquí contradice esa página o `reference/openapi.yaml`, confía en ellas.

## Cuándo usar este archivo

Cuando el integrador quiera:

- Probar su integración sin gastar créditos ni tener documentos reales.
- Provocar casos que en producción no puede elegir: SIGER o el SAT que no encuentran a la empresa, o que fallan.
- Probar su receptor de webhooks o su manejo de respuestas parciales.
- Saber cómo distinguir «no encontrado» de «error técnico».

## Los tres RFC

| RFC            | Escenario           | Qué prueba                                                             |
| -------------- | ------------------- | ---------------------------------------------------------------------- |
| `TRB010101OK1` | A · Todo encontrado | Camino feliz e información reconciliada. 22 ítems.                     |
| `TRB010101NF1` | B · No encontrado   | Resultado definitivo: las fuentes responden y no encuentran. 17 ítems. |
| `TRB010101ER1` | C · Error técnico   | Documentos ilegibles y consultas que fallan. 17 ítems.                 |

Una verificación de prueba **no consulta fuentes externas** (SAT, SIGER, INE, RENAPO), **no se cobra** y queda etiquetada con `prueba-trebol`.

## Cómo crearla

`POST /verifications` normal, con la API key de la cuenta de prueba. Manda el cuerpo completo: `country`, `tag`, `tax_id` con uno de los RFC y al menos un ítem válido.

```bash
curl -X POST "https://api.gotrebol.com/verifications" \
  -H "x-api-key: $TREBOL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "country": "mx",
    "tag": "prueba-escenario-a",
    "tax_id": "TRB010101OK1",
    "items": [
      { "type": "ac_mx", "options": { "file_url": "https://example.com/acta.pdf" } }
    ]
  }'
```

- Los ítems usan la llave `type`, no `item_type`.
- Trébol valida el cuerpo igual que en producción y después **reemplaza los ítems por los del escenario**. La respuesta `201` trae 22 o 17 ítems en `pending`.
- `country` queda en `mx` aunque se mande otro valor.
- Los archivos enviados se ignoran sin fallar. No hace falta subir nada a las `upload_url`: cada ítem documental se resuelve con un PDF de ejemplo.
- El mismo RFC devuelve la misma respuesta cada vez; solo cambian los `id` y las URL firmadas.

## Avanza en el tiempo

La verificación **no nace terminada**. Los ítems se completan escalonados y la verificación llega a `finished` unos 2 minutos y medio después:

| Segundos  | Ítems                                       |
| --------- | ------------------------------------------- |
| 10 – 30   | `csf_mx`, `proof_address`, `bank_statement` |
| 35 – 55   | `person_id`, `fme_mx`, `pw_mx`              |
| 60 – 90   | `aa_mx` ×2, `ac_mx`                         |
| ~100      | `siger`                                     |
| 110 – 120 | `public_sat_signatures` ×2                  |

Los tiempos reales llevan entre 5 y 15 segundos de desfase por el encolado.

Recién creada, `GET /v2/verifications/{verification-id}/shareholders` responde `shareholders_unavailable_reason: "pending_extraction"`. `external_identities` (RENAPO) aparece unos 20 segundos después del documento que aporta la CURP.

Los webhooks configurados en la cuenta notifican cada cambio igual que en una verificación real.

## Qué devuelve cada escenario

**A · Todo encontrado**

- SIGER `internal_status: "siger_documents_found"`; SAT `status: "success"` con certificados; INE `valid_id` en las credenciales `ine_mx`.
- `shareholders`: dos accionistas (persona moral `IDE150320AB3` con 60 %, persona física con 40 %), cada uno con sus documentos reconciliados en `identity`, `fiscal` y `address`.
- `people`: 5 en `key_people` (presidente, administrador único con `event: "removal"`, secretario, vocal, apoderado) y 7 en `full_list`. Seis personas con `external_identities.curp.success: true`; el titular del pasaporte no, porque no tiene CURP.

**B · No encontrado** (diferencias contra A)

- 17 ítems: no están los cinco documentos de los accionistas.
- `siger`: `item_internal_status: "siger_not_found"`, `item_value: {}`.
- `public_sat_signatures`: `status: "fail"`, `validation_result.reason: "rfc_not_found"`, `certificates: []`.
- `person_id` con `ine_mx`: `ine_validation_result: "invalid_id"` (un resultado, no un error). El pasaporte y la residencia siguen en `null`, como en A: la validación del INE solo corre con `ine_mx`.
- `shareholders: []` con `shareholders_unavailable_reason: "business_type_without_shareholders"`: es una sociedad cooperativa.
- `key_people: []`; las personas siguen en `full_list`. Ninguna trae `external_identities`.

**C · Error técnico** (diferencias contra A)

- `ac_mx` y `csf_mx` de la empresa: `item_error: "get_input_file_info_failed"`, `item_internal_status: null`, `item_value: {}`, sin documento. `documents_status: "partial_upload"`.
- `siger`: `item_internal_status: "siger_not_found_ops_forced"`.
- `public_sat_signatures`: `item_internal_status: "sat_scrap_failed"`, `validation_result.reason: "scraper_error"`.
- `person_id` con `ine_mx`: `ine_validation_result: "maximum_retries_reached"`. El pasaporte y la residencia siguen en `null`, como en A.
- `key_people`: 4 personas; el administrador único desaparece porque su rol venía del acta que falló. `details.registration` y `details.tax` vacíos.
- La verificación **sí llega a `finished`**.

## Distinguir «no encontrado» de «error técnico»

| Fuente     | No encontrado (B)                           | Error técnico (C)                           |
| ---------- | ------------------------------------------- | ------------------------------------------- |
| SIGER      | `siger_not_found`                           | `siger_not_found_ops_forced`                |
| SAT        | `validation_result.reason: "rfc_not_found"` | `validation_result.reason: "scraper_error"` |
| INE        | `invalid_id`                                | `maximum_retries_reached`                   |
| Documentos | sin `item_error`                            | `item_error: "get_input_file_info_failed"`  |

En SIGER `item_value` llega vacío en los dos casos: la única diferencia está en `item_internal_status`. Además de `siger_not_found_ops_forced`, también indican falla técnica `siger_error`, `siger_scrap_failed` y `siger_documents_scrap_failed`. El catálogo completo por fuente está en `reference/errors.md`.

Reglas para el integrador:

1. **La API responde `200` aunque la consulta falle**, y `item_status` es `complete` en ambos casos (solo toma `pending` o `complete`). El indicador vive dentro del ítem.
2. **No encontrado** → definitivo. Pedir al usuario que corrija el RFC o cargue otro documento.
3. **Error técnico** → crear el ítem de nuevo con `PUT /verifications/{verification-id}/add-items`.
4. **No se puede reintentar la lectura del mismo ítem**: Trébol ya agotó sus reintentos internos antes de emitir el estado.
