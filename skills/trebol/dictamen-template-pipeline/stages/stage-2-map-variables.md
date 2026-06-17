---
name: stage-2-map-variables
description: Segunda etapa del pipeline de configuración de plantillas de dictamen. Toma los campos del cliente extraídos por Stage 1 y produce un MAPEO CANDIDATO campo→variable de Trébol, fundado en el diccionario de variables (grounding), la plantilla de referencia, y el ejemplo diligenciado si existe. Cada campo recibe una variable propuesta, o SIN_VARIABLE / AMBIGUO con nota, más confianza y racional corto. Úsalo cuando el orquestador lo invoque como segundo paso. Es un borrador mecánico, no definitivo — existe para que el panel (Stage 3) lo critique. No participa en la discusión.
---

# Stage 2 — Map Variables (Mapeo candidato)

Segunda etapa. Produce un **primer mapeo** de cada campo del cliente a una variable de Trébol. Es un borrador fundamentado, no la decisión final.

## Responsabilidad

**Una sola cosa:** por cada campo de Stage 1, proponer la variable de Trébol más probable (o marcar que no hay / es ambigua), con confianza y racional. No discute, no decide: eso es Stage 3.

Mantener el borrador (este stage) separado de la crítica (Stage 3) es intencional — evita que quien propone defienda complacientemente su propia propuesta.

## Grounding que cargas
- `grounding/dictionary-example.json` (+ `dictionary-example.md`) — **payload real de variables**: fuente primaria para confirmar la **grafía exacta** de una variable y su **forma** (escalar → variable simple; lista → bucle/instancias; objeto → sub-campos). Consúltalo programáticamente (grep/python), no lo cargues entero. Si una variable está aquí, así se escribe. Recuerda el caveat: es un payload de UNA verificación, la ausencia no descarta una variable.
- `grounding/variable-catalog.md` — el universo de variables permitidas (incluye familias que el payload puede no tener).
- `grounding/sintaxis-plantillas.md` — para decidir si un campo se expresa como variable simple, bucle o inverso.
- `grounding/example-template-reference.md` — patrones probados sección por sección. Si la plantilla del cliente tiene una sección análoga a la referencia, **reusa el patrón de la referencia**.
- `corrections.md` — máxima prioridad ante conflicto.

## Proceso

### Paso 1 — Resolver país/familia
Según `country` de Stage 1, restringe el universo: México → familias `legal_*/tax_*/key_person_*/shareholder_*/board_member_*/executive_*/auditor_*/external_sources_*/ubos_business_*`; Colombia → `rut_co/cc_co_ops/rues/public_*/ubos_*`. No propongas variables de la familia equivocada.

### Paso 2 — Mapear cada campo
Para cada campo, en este orden de evidencia:
1. **Ejemplo diligenciado** (si existe): el `example_value` casi determina la variable (ej. un RFC → `{tax_businessTaxId}`; una fecha de constitución → `{registration_registrationDate}`).
2. **Coincidencia con la plantilla de referencia**: si la etiqueta y la sección equivalen a una sección de la referencia, copia el token de la referencia.
3. **Coincidencia semántica de etiqueta** contra el catálogo (ej. "DURACIÓN" → `{legal_businessDuration}`; "DOMICILIO FISCAL" → `{fiscalAddress}`).
4. **Confirma grafía y forma contra `dictionary-example.json`**: una vez que tienes la variable candidata, verifica en el payload real que la clave existe con esa grafía exacta y qué forma tiene su valor (escalar/lista/objeto). Si el valor es lista → es bucle/instancias; si es escalar → variable simple. Si la candidata está en el JSON, sube la confianza; si no está pero sí en catálogo/referencia, mantén la candidata pero recuerda que la ausencia en el payload no la descarta.

Decide la forma según `kind` y `has_loops_capable`:
- `single` → variable simple.
- `table_row` en Word → bucle `{#lista}…{/}` con variables relativas.
- `table_row` en PDF → **no hay bucle**: propón variables indexadas para las primeras N instancias conocidas (`key_person_1`, `key_person_2`, … / `shareholder_0`, `shareholder_1`, …) y marca `pdf_repeat_limit` para que Stage 3 decida cuántas instancias configurar.
- `source_clause` → patrón condicional+inverso de FUENTE (ver referencia).
- `checkbox_like` → variable `*_x_mark`.

### Paso 3 — Asignar resultado por campo
- `variable`: el/los token(s) propuestos, ya con la sintaxis correcta.
- `confidence`: `alta` (ejemplo diligenciado, referencia exacta, o variable confirmada en `dictionary-example.json`) | `media` (etiqueta clara) | `baja` (inferencia).
- `status`: `MAPEADO` | `AMBIGUO` (varias variables posibles — lístalas) | `SIN_VARIABLE` (no existe variable para este campo).
- `rationale`: 1–3 frases. Obligatorio. Cita la fuente de evidencia (ejemplo/referencia/catálogo).
- `flags`: numeración (1-based vs 0-based), poder `delegate` no documentado, separador `_`/`-` Colombia, mayúsculas, espacios en llaves, etc. — cualquier gotcha del grounding que aplique.

### Paso 4 — No inventar
Si un campo no tiene variable en el catálogo/referencia/web, **`SIN_VARIABLE`** con nota. No "deduzcas" un ID plausible. Ejemplos de campos sin variable habituales: datos que el cliente pide pero Trébol no extrae (notas internas, folios propios del cliente, firmas manuscritas, sellos).

### Paso 5 — Verificar existencia
Cada variable propuesta debe existir literalmente en el grounding. Si no estás seguro de que exista (ej. una variante de domicilio), márcala `confidence: baja` y `flags: verificar_existencia` para que Stage 3 la valide (incluso consultando la doc web).

## Output (markdown para Stage 3)

```
# Candidate Mapping
## F001  "DENOMINACIÓN O RAZÓN SOCIAL"
- variable: {legal_businessName}, {legal_businessType}
- form: single
- status: MAPEADO
- confidence: alta
- flags: []
- rationale: "Ejemplo diligenciado mostraba 'GRUPO EJEMPLO, S.A. DE C.V.'; coincide con patrón de la referencia (DATOS GENERALES)."

## F014  "FACULTADES DEL APODERADO"
- variable: {key_person_1_power_administration_x_mark} ... (6 poderes)
- form: checkbox_like
- status: MAPEADO
- confidence: media
- flags: [numeracion_1based, poder_delegate_no_documentado]
- rationale: "Reusa el patrón de poderes de la referencia. Apoderados son 1-based. La referencia usa 'delegate', no documentado en web — marcar para verificación."

## F033  "FIRMA Y SELLO DEL REPRESENTANTE"
- variable: —
- status: SIN_VARIABLE
- rationale: "Trébol no extrae firmas manuscritas ni sellos; dejar como campo fijo en blanco."
```

## Reglas duras
1. **No inventes variables.** Solo del grounding/web. Sin coincidencia → `SIN_VARIABLE`.
2. **Familia correcta por país.** Nada de `legal_*` en plantilla Colombia ni `rut_co` en México.
3. **Respeta forma vs capacidad.** Bucles solo si `has_loops_capable`. En PDF, indexa.
4. **Numeración correcta.** Apoderados 1-based; accionistas/órganos 0-based. Marcar en `flags`.
5. **Racional siempre, citando evidencia.**
6. **No decidas desacuerdos.** Si hay dos variables plausibles, `AMBIGUO` con ambas; Stage 3 decide.

## Manejo de errores
- **Campo ya tokenizado por el cliente:** valida que la variable exista; si está mal escrita (espacio en llaves, mayúsculas), propón la corrección y marca `flags: corregir_token_cliente`.
- **Sin ejemplo diligenciado:** apóyate en referencia + etiqueta; sube la proporción de `AMBIGUO` en vez de adivinar.
- **Sección no cubierta por el catálogo** (ej. una consulta externa nueva): márcala `SIN_VARIABLE` con `flags: posible_variable_no_catalogada` para que Stage 3 consulte la doc web.

## Checklist
- [ ] Todos los campos de Stage 1 tienen una entrada.
- [ ] Cada variable propuesta existe en el grounding (o está marcada para verificar).
- [ ] Forma (simple/bucle/inverso/x_mark) coherente con tipo de archivo.
- [ ] Numeración y gotchas marcados en `flags`.
- [ ] `SIN_VARIABLE` usado en vez de inventar.
- [ ] Racional con evidencia en cada campo.
