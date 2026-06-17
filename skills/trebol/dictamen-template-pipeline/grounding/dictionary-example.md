# Diccionario de Variables — Payload de Ejemplo Real (`dictionary-example.json`)

Este es un **payload real de variables** exportado de una verificación de la plataforma de Trébol. Es el grounding más autoritativo que tenemos sobre **cómo se llaman exactamente las variables** y **qué forma tienen los datos** que Trébol inyecta en una plantilla.

## Qué es exactamente

- Un **mapa plano** JSON: `nombre_de_variable → valor de ejemplo`.
- **3.420 claves** en total. ~469 son plumbing de FUENTE (`*_source*`); el resto son variables de dato. Hay ~1.176 "stems" lógicos distintos (colapsando los índices numéricos).
- Su base proviene de **una verificación de ejemplo**, una empresa mexicana (`CONSTRUCTORA DEMO, S.A. DE C.V.`, una sociedad unipersonal). Por eso casi todos los valores son datos de ejemplo de esa empresa.
- **Enriquecido con un segundo payload de ejemplo** (`DISTRIBUIDORA EJEMPLO, S.A. DE C.V.`, otra verificación MX): al comparar ambos, el segundo resultó cubrir casi exactamente las mismas familias que el primero, **salvo 4 sub-campos de domicilio desagregado** que el primero no traía. Esos 4 se añadieron al JSON para completar el esquema de direcciones (ver "Direcciones" abajo). El resto del segundo payload ya estaba cubierto, así que no se duplicó.

## Para qué sirve (y para qué NO)

**Sí es autoritativo para:**
- El **nombre exacto** de cada variable presente (grafía, mayúsculas/minúsculas, separadores, numeración). Si una variable aparece aquí, así se escribe.
- La **forma del dato**: si es texto, número, booleano, lista (bucle) u objeto anidado. Esto le dice al ingeniero y al legal qué esperar en la plantilla.
- Confirmar la **numeración por familia** con datos reales (ver abajo).

**NO es un esquema exhaustivo.** Es el payload de *una* empresa. Las familias que **no aparecen aquí no están descartadas** — simplemente esta empresa no las tenía en su extracción. Ejemplos: en este payload **no hay** `key_person_*` (apoderados) ni `board_member_*` (consejo), porque esa sociedad unipersonal no los tenía. Eso **no significa** que esas variables no existan; siguen documentadas en `variable-catalog.md`, `example-template-reference.md` y la doc web.

> Regla: **la presencia confirma; la ausencia no descarta.** Si una variable está en el JSON, úsala con confianza. Si no está, cae a los otros documentos de grounding antes de concluir que no existe.

## Estructura de los valores

Los valores vienen en cuatro formas, y cada una le dice al pipeline cómo tokenizar:

- **Escalar** (string / int / bool / float) → variable simple `{variable}`.
  Ej.: `"legal_businessName": "CONSTRUCTORA DEMO"`, `"is_unipersonal": true`, `"capital_fixedValue": "50.000"`.
- **Lista** → familia de **bucle** `{#lista}...{/}` (en Word) o instancias indexadas (en PDF).
  Ej.: `sources`, `actaSources`, `external_sources_renapo`, `executives`, `auditors`, `shareholders`. Cada elemento de la lista es un objeto con las sub-claves que van **relativas dentro del bucle**.
- **Objeto anidado** (dict) → grupo de sub-campos accesibles por sufijo.
  Ej.: `"renapo_0"` es un objeto con `curp`, `names`, `first_surname`, …; en las plantillas eso se ve aplanado como `renapo_0_curp`, `renapo_0_names`, etc. (ambas formas conviven en el JSON).
- Las claves `*_source*` son el plumbing de **FUENTE** (escritura, fecha, notario, folio). Confirman cómo se nombra cada pieza de la cláusula de FUENTE.

## Familias presentes en este payload

Las más voluminosas: `sources` (981), `shareholder` (624), `actaSources` (529), `fiel`/`fiel_sello` (211), `executive` (147), `people` (147), `siger` (130), `legal` (114), `ubos` (84), `renapo` (81), `tax` (75), `ine` (74), `registration` (73), `auditor` (45), más `capital`, `forms`, `comercialAddress`/`fiscalAddress` (+ sus `*Disaggregated`), `aml_validation_*`, `verification_*`, `group_*`, `share_*`.

Esto confirma con datos reales varias familias que en `variable-catalog.md` estaban solo documentadas: las consultas externas (RENAPO, INE, SIGER, FIEL/Sello), `ubos_*`, `tax_*`, `external_sources_*`, y el plumbing completo de FUENTE.

## Lo que este payload confirma (verificado contra dato real)

- **Numeración 0-based** confirmada para `shareholder_0`/`shareholder_1`, `executive_0`, `renapo_0`/`1`/`2`, `auditor_0`. Consistente con la regla del catálogo (accionistas/órganos 0-based).
- **`tax_additionalData`** se escribe **con la "i"** (`additionalData`), confirmado por las claves reales `tax_additionalData`, `tax_additionalData_source_itemId`. La grafía `tax_additonalData` que aparece en la plantilla de referencia es una **errata** y no debe replicarse. (Esto se reflejó en `corrections.md`.)
- **Sin claves con guion medio** en este payload (todo `_`). No resuelve la duda de Colombia (este payload es de México), pero confirma que la familia mexicana usa `_` de forma consistente.

## Direcciones (comercial vs fiscal) — esquema completo

Comparar los dos payloads confirmó algo importante para configurar plantillas: **cada domicilio de la empresa viene en dos formas que conviven** (línea única + desagregado), y **las dos direcciones desagregadas NO usan el mismo esquema de sub-llaves**:

- **Comercial desagregada → sub-llaves en ESPAÑOL** (`comercialAddressDisaggregated_*`): `tipo_vialidad`, `nombre_vialidad`, `numero_exterior`, `tipo_asentamiento`, `nombre_asentamiento`, `codigo_postal`, `municipio_o_ente_territorial`, `entidad_federativa`, `pais`. Más `comercialAddress` (línea única), `comercialAddress_razon_social` y `comercialAddress_source_date(+_day/_month/_year)`.
- **Fiscal desagregada → sub-llaves en INGLÉS** (`fiscalAddressDisaggregated_*`): `street_type`, `street_name`, `external_number`, `internal_number`, `neighborhood`, `district`, `post_code`, `state`. Más `fiscalAddress` (línea única) y `fiscalAddress_source_date(+_day/_month/_year)`.

No existe `fiscalAddressDisaggregated_codigo_postal` ni `comercialAddressDisaggregated_post_code`. Esta asimetría es **intencional** (no es un typo a "corregir").

Las 4 sub-llaves que el segundo payload aportó y se añadieron al JSON: `comercialAddressDisaggregated_tipo_vialidad`, `comercialAddressDisaggregated_nombre_vialidad`, `comercialAddressDisaggregated_tipo_asentamiento`, `fiscalAddressDisaggregated_street_type`.

Regla de mapeo: si la plantilla del cliente tiene **una sola línea** de dirección → usa `{comercialAddress}` / `{fiscalAddress}`. Si **separa** la dirección en campos → usa los sub-campos desagregados eligiendo el idioma por familia (español para comercial, inglés para fiscal). El detalle completo, incluidos los domicilios de personas (`*_mx_address_*`), está en `variable-catalog.md` → sección "Domicilios".

## Cómo usar este archivo en el pipeline (importante)

El archivo pesa ~892 KB. **No lo cargues entero en contexto.** Consúltalo **programáticamente** cuando un stage necesite verificar una variable:

```bash
# ¿Existe esta variable? ¿Cómo se escribe exactamente?
python3 -c "import json; d=json.load(open('grounding/dictionary-example.json')); print([k for k in d if 'shareholder' in k][:20])"

# ¿Qué forma tiene el dato (escalar/lista/objeto)?
python3 -c "import json; d=json.load(open('grounding/dictionary-example.json')); v=d['actaSources']; print(type(v).__name__, (v[0] if isinstance(v,list) and v else v))"
```

- **Stage 2 (Map Variables)** lo usa como fuente principal para confirmar la grafía exacta de cada variable candidata y decidir simple vs bucle según la forma del valor.
- **Stage 3 (ingeniero)** lo cita como prueba de existencia/escritura ("la clave real es `X`, no `Y`").
- **Stage 5 (helpers)** lo consulta para resolver si una familia es lista (bucle/instancias) o escalar.

## Orden de precedencia del grounding

1. `corrections.md` — máxima prioridad (verdad confirmada contra verificaciones reales).
2. **`dictionary-example.json`** (este archivo) — ground truth de nombres y formas para las familias presentes.
3. `example-template-reference.md` — cómo se ve una plantilla bien tokenizada (pero tiene erratas conocidas).
4. `variable-catalog.md` — explicación destilada y catálogo curado (incluye familias ausentes en el JSON).
5. Documentación viva en `gotrebol.com/docs/plantillas/` — referencia general.

Ante conflicto entre el JSON y el catálogo/referencia sobre **cómo se escribe** una variable presente en el JSON, **gana el JSON** (es dato real), salvo que `corrections` diga otra cosa.
