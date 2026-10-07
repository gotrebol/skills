# RFC de prueba — Verificaciones KYB México con respuesta fija

Fuente canónica: `guia-devs/pruebas/rfc-de-prueba.mdx` (https://docs.gotrebol.com/guia-devs/pruebas/rfc-de-prueba). Si algo aquí contradice esa página o `reference/openapi.yaml`, confía en ellas.

## Cuándo usar este archivo

Cuando el integrador quiera:

- Probar su integración sin gastar créditos ni tener documentos reales.
- Provocar casos que en producción no puede elegir: SIGER o el SAT que no encuentran a la empresa, o que fallan.
- Probar su receptor de webhooks o su manejo de respuestas parciales.
- Saber cómo distinguir «no encontrado» de «error técnico».
- Ver cómo llega un `doc_validation` que no pasa sus reglas o que recibe un documento de otro tipo.
- Probar solo la consulta de RFC (SAT y SIGER), sin documentos.

## Los RFC

| RFC            | Escenario                    | Qué prueba                                                                                          |
| -------------- | ---------------------------- | --------------------------------------------------------------------------------------------------- |
| `TRB010101OK1` | A · Todo encontrado          | Camino feliz e información reconciliada. 22 ítems.                                                  |
| `TRB010101NF1` | B · No encontrado            | Resultado definitivo: las fuentes responden y no encuentran. 17 ítems.                              |
| `TRB010101ER1` | C · Error técnico            | Documentos ilegibles y consultas que fallan. 17 ítems.                                              |
| `TRB010101DV1` | D · Validación de documentos | `doc_validation` con reglas que pasan, reglas que no pasan y un documento de otro tipo. 4 ítems.    |
| `TRB010101RF1` | E · Consulta de RFC          | Solo consultas públicas: SAT de la empresa y de su representante, SIGER y empresas relacionadas. 4 ítems. |

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
- Trébol valida el cuerpo igual que en producción y después **reemplaza los ítems por los del escenario**. La respuesta `201` trae 22 (A), 17 (B y C) o 4 (D y E) ítems en `pending`.
- `country` queda en `mx` aunque se mande otro valor.
- Los archivos enviados se ignoran sin fallar. No hace falta subir nada a las `upload_url`: cada ítem documental se resuelve con un PDF de ejemplo.
- El mismo RFC devuelve la misma respuesta cada vez; solo cambian los `id` y las URL firmadas.

## Avanza en el tiempo

La verificación **no nace terminada**. En A, B y C los ítems se completan escalonados y la verificación llega a `finished` unos 2 minutos y medio después (D en ~1 minuto y E en ~45 s; ver sus secciones):

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

**D · Validación de documentos** (no se compara con A)

- 4 ítems `doc_validation`, cada uno con `allowed_item_types` y un `ruleset`. Se completan a los ~15, 30, 45 y 60 s.

| Esperado         | Regla                                | Resultado                                                       | `item_internal_status` |
| ---------------- | ------------------------------------ | --------------------------------------------------------------- | ---------------------- |
| `csf_mx`         | `vr_trebol_antiguedad` (60 días)     | Regla no cumplida: emitida el 2026-01-10                        | `validation_failed`    |
| `ac_mx`          | `accionistas_rule` (personalizada)   | Pasa                                                            | `validation_success`   |
| `person_id`      | `vr_trebol_vencimiento`              | Regla no cumplida: vigente hasta 2025                           | `validation_failed`    |
| `bank_statement` | `vr_trebol_antiguedad` (60 días)     | Tipo incorrecto (es un recibo de CFE); la regla sí se cumple    | `validation_failed`    |

- Resultados fijos, evaluados el 2026-10-01: no se recalculan con la fecha del día. El recibo del 2026-08-05 cumple los 60 días (tenía 57); la constancia del 2026-01-10 no.
- **No hay `item_error`** en los que fallan. El motivo está en `item_value`: `item_type_validation_result[0].validation_result: false` si el tipo no coincide; si coincide, la regla con `validation_result: false` en `rules_validation_result`.
- Solo las reglas personalizadas devuelven `validation_rule` con su texto; las `vr_trebol_*` solo traen `id`, `variable_value` y `validation_result`.
- Los cuatro quedan en `item_status: "complete"` y la verificación llega a `finished`. No se crean ítems tipados a partir de los que pasan; `details`, `people` y `shareholders` llegan vacías.

**E · Consulta de RFC** (no se compara con A)

- 4 ítems, sin documentos: `public_sat_signatures` `business` (~10 s), `public_sat_signatures` `representatives` (~45 s), `siger` (~120 s) y `siger_shareholders` (~200 s). Todos con éxito.
- La verificación llega a `finished` a los ~45 s, con los dos SAT: no espera a `siger` (sin `siger_data_extraction`) ni a `siger_shareholders` (`is_optional`), que se completan después, igual que en producción. No dar SIGER por ausente solo porque la verificación ya está en `finished`.
- SAT de la empresa: sello y FIEL activos, con `legalRepresentativeRFC: "RUTD800515AB2"`, más su historial caduco o revocado.
- SAT de representantes: consulta el RFC que nombran los certificados de la empresa, así que `tax_ids` es `["RUTD800515AB2"]`, no el RFC de la empresa.
- `siger`: `siger_documents_found`, folio `N-2024095979`, seis actos con `fme_assembly_minute_extractor` y dos socios. Los actos no traen `document_url` y no se crean actas.
- `siger_shareholders`: `siger_shareholders_scrap_completed`, con las empresas relacionadas de cada socio. Los socios llegan como SIGER los devuelve: nombre completo en `paternal_surname` y `type: "individual"`, aunque sean sociedades.
- `details`: solo `legal.business_folio_number`, cuando `siger` ya se completó. `people` y `shareholders` vacías (`shareholders_unavailable_reason: null`).

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
