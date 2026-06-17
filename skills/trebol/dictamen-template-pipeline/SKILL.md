---
name: dictamen-template-pipeline
description: Orquesta el pipeline completo para CONFIGURAR plantillas de dictamen de clientes de Trébol — es decir, tomar la plantilla en blanco que envía un cliente (Word .docx o PDF editable) y devolverla idéntica en formato pero con las variables/IDs de Trébol (ej. {legal_businessName}, {#shareholders}...{/}) insertadas en los lugares correctos, para que la plataforma de Trébol pueda autollenarla. Úsalo siempre que el usuario suba o mencione una plantilla de dictamen, un "formato de dictamen", un "formato de opinión legal", una plantilla de cliente que hay que "configurar", "parametrizar", "mapear variables", "ponerle los IDs", o pregunte "¿cómo configuro esta plantilla?" adjuntando un Word o PDF. Dispara también si menciona "plantillas de formatos", "exportación de verificaciones", "diccionario de variables", o pide acelerar la configuración de plantillas. NO es para responder cuestionarios de seguridad (eso es el security-questionnaire-pipeline).
---

# Dictamen Template Pipeline — Orquestador

Este skill coordina el pipeline end-to-end para **configurar plantillas de dictamen** de clientes de Trébol. No contiene lógica de dominio — solo orquesta. Toda la inteligencia especializada vive en los archivos de referencia `.md` bajo `stages/`, `personas/`, `helpers/`, en `corrections.md` y en `grounding/`, que este orquestador lee y aplica paso a paso.

## Qué significa "configurar una plantilla" (leer esto primero)

El cliente entrega una plantilla de dictamen **en blanco** (un `.docx` o un PDF editable) con su propio diseño, encabezados, tablas y etiquetas. "Configurar" significa:

> Devolver **el mismo archivo, en el mismo formato**, con las **variables de Trébol** (ej. `{legal_businessName}`, `{tax_businessTaxId}`, `{#shareholders}{name}{/}`) colocadas en los lugares correctos, de modo que cuando Trébol corra ese archivo contra una verificación a través del endpoint de exportación (`GET` o `POST /v2/verifications/{verification-id}/export/{doc-template-id}`, ver `../reference/openapi.yaml`), devuelva un `download_url` del documento ya relleno con los datos extraídos.

No se redacta contenido nuevo, no se reescribe el dictamen, no se cambia el diseño. Solo se **mapean los campos del cliente a variables de Trébol** y se **insertan los tokens** preservando el formato exacto.

- **Word (.docx):** los tokens van como texto inline: `{variable}`, bucles `{#lista}...{/}`, condicionales/inversos `{^variable}...{/}`. Los bucles **solo funcionan en Word**.
- **PDF editable:** los tokens van como **valor del campo de formulario** (Convención A, confirmada en producción: el nombre del campo es arbitrario; si la plantilla ya nombra los campos como variables, respeta esa convención). Un token por campo; nunca duplicar. Los bucles **no funcionan en PDF**.

## Cuándo disparar

Dispara este skill cuando detectes cualquiera de estos indicadores:

- El usuario sube un Word o PDF que parece una plantilla de dictamen / formato de opinión legal en blanco (con etiquetas como "DENOMINACIÓN O RAZÓN SOCIAL", "APODERADOS", "ACCIONISTAS", "CONSEJO DE ADMINISTRACIÓN", "FUENTE", etc.) y pide configurarla / parametrizarla / ponerle las variables.
- El usuario menciona "plantilla de dictamen", "formato de dictamen", "plantilla de cliente", "configurar plantilla", "mapear variables", "diccionario de variables", "plantillas de formatos", "exportación de verificaciones".
- El usuario adjunta dos archivos: una plantilla en blanco + un ejemplo ya diligenciado, y pide replicar la configuración.

Si hay duda razonable sobre si aplica, dispáralo — Stage 1 determinará si el input es realmente una plantilla configurable y pedirá clarificación si no lo es.

**No dispares** para cuestionarios de seguridad de la información de bancos: eso es `security-questionnaire-pipeline`, otro pipeline.

## Arquitectura (sub-pipeline dentro del skill de Trébol)

Este pipeline vive dentro de `skills/trebol/` y es invocado desde el `SKILL.md` padre (`../SKILL.md`). El asistente llega aquí porque el skill de Trébol lo referencia en su sección "Plantillas de dictamen" — no se instala por separado. Todo lo demás en esta carpeta son **archivos de referencia `.md`** que este orquestador **lee y sigue** cuando llega a cada paso; no son skills independientes.

```
skills/trebol/
├── SKILL.md                    ← entrada del skill de Trébol (padre, instalado por npx skills add)
└── dictamen-template-pipeline/
    ├── SKILL.md                ← este orquestador (sub-pipeline, referenciado desde el padre)
├── corrections.md              ← correcciones conocidas (máxima prioridad)
├── stages/                     ← 6 etapas del proceso (un .md por etapa)
│   ├── stage-1-parse-template.md
│   ├── stage-2-map-variables.md
│   ├── stage-3-panel-discussion.md
│   ├── stage-4-user-qa.md
│   ├── stage-5-configure-deliverable.md
│   └── stage-6-rationale-packaging.md
├── personas/                   ← 3 perspectivas de reviewer (un .md cada una)
│   ├── persona-trebol-cpo-reviewer.md
│   ├── persona-trebol-template-engineer.md
│   └── persona-trebol-legal-dictamen.md
├── helpers/                    ← configuradores de Word y de PDF
│   ├── helper-docx-template-filler.md
│   └── helper-pdf-form-configurator.md
    └── grounding/              ← diccionario de variables + sintaxis + plantilla de referencia + payload real
```

> **Cómo opera el orquestador:** cuando un paso dice "lee y aplica `stages/stage-1-parse-template.md`", abre ese archivo con la herramienta de lectura y sigue sus instrucciones como si fueran una sección de este mismo documento. Las "personas" se adoptan leyendo su `.md` y razonando desde esa perspectiva. No se invoca ninguna skill externa (salvo las públicas de `docx`/`pdf` que los helpers referencian como fallback).

## Flujo end-to-end

El pipeline es **estrictamente secuencial** con un único punto de pausa humana (Stage 4, solo cuando hace falta). No saltar stages.

```
Inputs → [1 Parse Template] → [2 Map Variables] → [3 Panel Discussion]
       → [4 User Q&A] ⏸  PAUSA (solo si hay campos sin resolver)
       → [5 Configure Deliverable] → [6 Rationale Packaging] → Output
```

Mapeo mental con el pipeline de cuestionarios que el equipo ya conoce: 1≈Intake, 2+3≈Panel/Discusión, 4=mismo checkpoint humano, 5≈Draft+helpers, 6=mismo packaging con racional.

### Paso 0 — Preparación (antes de aplicar Stage 1)

1. Identifica los archivos de input:
   - **Plantilla del cliente** (obligatoria): `.docx` o PDF editable que se va a configurar.
   - **Ejemplo diligenciado** (opcional, "no siempre"): una plantilla ya llena que muestra qué dato va en cada campo. Si está, es oro: úsalo para mapear con alta confianza.

2. **Carga el grounding (diccionario de variables).** Lee en silencio — no lo listes al usuario, solo internalízalo:
   - `grounding/dictionary-example.md` + `grounding/dictionary-example.json` — **payload real de variables** exportado de una verificación de la plataforma (mapa `variable → valor de ejemplo`, ~3.416 claves). Es la **fuente de verdad de cómo se llama exactamente cada variable y qué forma tiene el dato** (escalar / lista-bucle / objeto). **No cargues el `.json` entero en contexto** (pesa ~892 KB) — consúltalo programáticamente (grep/python) cuando un stage necesite confirmar una variable. Lee primero el `.md` companion: explica la estructura, el caveat clave (es un payload de UNA verificación mexicana, así que **la presencia confirma pero la ausencia no descarta** una variable), y el orden de precedencia. Stage 2/3/5 lo usan como fuente primaria de grafía y forma.
   - `grounding/variable-catalog.md` — catálogo destilado de TODAS las variables de Trébol (empresa, apoderados, accionistas, administración, vigilancia, Colombia, y las "consultas adicionales": RENAPO, PLD, FIEL/Sello, SIGER, INE, QR-CSF, UBOs MX). Incluye familias que pueden no aparecer en el JSON de ejemplo (ej. apoderados/consejo si esa verificación no los tenía).
   - `grounding/sintaxis-plantillas.md` — sintaxis del motor de plantillas (variables, bucles `{#}{/}`, inversos `{^}{/}`, numeración, x_mark, fuentes) y las diferencias Word vs PDF.
   - `grounding/example-template-reference.md` — la plantilla de dictamen de referencia de Trébol, ya tokenizada, anotada sección por sección. Es la fuente de verdad de **cómo se ve una plantilla bien configurada** y la única que documenta variables que no están en la web (ej. `ubos_business_0_*`, `external_sources_*`, `tax_additionalData_*`). Tiene erratas conocidas (ver `corrections.md`).
   - `grounding/example-client-mapping.md` — **ejemplo trabajado de una plantilla de cliente real ya configurada** (PDF de formulario). Documenta cómo se mapea un PDF real: el token va en el **valor** del campo (no en el nombre), el mapeo de etiquetas de poderes (incl. `delegate` confirmado y `dominio→assets_management`), las dos variantes de domicilio, los flags del comprobante (`address_service_type_*`), y los slots fijos de apoderados/consejo/UBOs. Léelo cuando configures PDFs o cuando dudes cómo se llena un campo.
   - La documentación viva está en `https://docs.gotrebol.com/plantillas/` (intro, empresa, apoderados, accionistas, administracion, vigilancia, colombia, configuracion, gestion). Si necesitas confirmar una variable que no esté en el catálogo local, consúltala ahí.
   - **Precedencia:** ante conflicto sobre cómo se escribe una variable, `corrections` > `dictionary-example.json` (dato real) > `example-template-reference.md` > `variable-catalog.md` > doc web.

3. **Carga el skill de correcciones conocidas.** Lee `corrections.md` en silencio.
   - Si está vacío, continúa sin él.
   - Si tiene entradas, aplícalas con **máxima prioridad**: ante cualquier conflicto entre una entrada de `corrections` y el catálogo, la documentación web o la plantilla de referencia, **prevalece `corrections`**.

4. Si no pudiste cargar ningún grounding, detente y avisa al usuario: el pipeline **no debe correr sin el diccionario de variables**, porque los actores inventarían IDs que no existen.

### Paso 1 — Stage 1: Parse Template

Lee y aplica `stages/stage-1-parse-template.md` con la plantilla del cliente (y el ejemplo diligenciado si existe). Output esperado: estructura normalizada de la plantilla — tipo de archivo, lista de **campos del cliente** (cada uno con su etiqueta literal, ubicación, si es campo único o fila repetible de tabla, formato esperado), país/jurisdicción detectado, y formato de entregable. Guarda como `state.template`.

Si Stage 1 reporta que el input no es procesable (no es una plantilla, archivo corrupto, PDF no editable sin campos), detente y devuelve ese mensaje. No continúes.

### Paso 2 — Stage 2: Map Variables

Lee y aplica `stages/stage-2-map-variables.md` con `state.template` y el grounding. Produce un **mapeo candidato**: por cada campo del cliente, la variable de Trébol propuesta (o `SIN_VARIABLE` / `AMBIGUO` con nota), confianza, y racional corto. Este mapeo es un borrador mecánico fundado en el catálogo — **no es definitivo**; existe para que el panel lo critique. Guarda como `state.candidate_mapping`.

Mantener separado el borrador (Stage 2) de la revisión (Stage 3) es intencional: evita que el ingeniero revise complacientemente su propio trabajo.

### Paso 3 — Stage 3: Panel Discussion

Lee y aplica `stages/stage-3-panel-discussion.md` con `state.template` y `state.candidate_mapping`. Ese módulo te hace adoptar, una por una, las 3 perspectivas de `personas/` (CPO, ingeniero de variables, legal de dictamen) para revisar el mapeo candidato y **discutir qué va en cada campo**, resolviendo desacuerdos. Output esperado: mapeo consolidado por campo con decisión (variable final / sin variable / pregunta al usuario), disensos registrados, y la lista de campos que requieren input humano. Guarda como `state.panel_decisions`.

### Paso 4 — Stage 4: User Q&A ⏸ PUNTO DE PAUSA (condicional)

Lee y aplica `stages/stage-4-user-qa.md` con el estado acumulado. Produce un **markdown breve** con solo los campos que el panel no pudo resolver (campos sin variable clara, ambigüedades, decisiones de negocio).

- Si **no hay** campos abiertos, este stage devuelve "sin dudas" y el pipeline avanza directo al Stage 5 sin pausar.
- Si **sí hay**, **detente aquí**, entrega el markdown y espera respuesta del usuario. No auto-rellenes, no inventes mapeos, no avances hasta tener respuestas explícitas. Guarda como `state.user_answers` y avanza. Si responde parcial, procesa lo respondido y vuelve a preguntar solo lo que falta.

### Paso 5 — Stage 5: Configure Deliverable

Lee y aplica `stages/stage-5-configure-deliverable.md` con todo el estado. Según el tipo de archivo, aplica el helper correspondiente:

- Plantilla Word (.docx) → `helpers/helper-docx-template-filler.md`
- Plantilla PDF editable → `helpers/helper-pdf-form-configurator.md`

El helper inserta los tokens en el archivo **preservando el formato exacto del cliente**. Guarda como `state.configured_template`.

### Paso 6 — Stage 6: Rationale Packaging

Lee y aplica `stages/stage-6-rationale-packaging.md` con `state.configured_template` y `state.panel_decisions`. Produce, además del archivo configurado listo para subir a Trébol:

- Un **mapa de variables** auditable (tabla campo del cliente → variable de Trébol).
- Un **racional corto** (3–6 bullets) con las decisiones clave.
- La lista de **campos sin variable** que quedaron como texto fijo o vacíos, y por qué.

Devuelve al usuario con `present_files`: el archivo configurado + el documento de mapa/racional.

## Contratos entre stages

- **Input:** cada stage recibe el estado acumulado como parámetros explícitos.
- **Output:** objeto estructurado (markdown con secciones claras) que el siguiente stage consume sin parsing ambiguo.
- **Racional siempre:** toda propuesta de variable, decisión o campo-sin-variable lleva racional de 1–3 frases. Invariante del pipeline.

Si un stage devuelve algo que no cumple el contrato, no lo arregles en el orquestador — detente y reporta que el stage falló.

## Reglas duras

1. **El formato del cliente es sagrado.** El archivo de salida es idéntico al de entrada salvo por los tokens insertados. No cambies fuente, tamaño, colores, anchos de columna, encabezados, ni el diseño. No apliques brand guidelines de Trébol sobre la plantilla del cliente.
2. **Nunca inventes variables.** Solo se usan variables que existan en `grounding/variable-catalog.md`, en la plantilla de referencia, o en la documentación web. Si un campo del cliente no tiene variable, se marca `SIN_VARIABLE` y se decide en Stage 3/4 — no se inventa un ID plausible.
3. **Nunca corras sin grounding.** El diccionario de variables debe estar cargado. Si `corrections` tiene entradas, prevalecen sobre todo lo demás.
4. **Respeta el punto de pausa.** Si hay campos sin resolver, el usuario decide antes del Stage 5. No auto-rellenes ambigüedades.
5. **No fusiones borrador y revisión.** Stage 2 propone, Stage 3 critica. Son pasos distintos a propósito.
6. **Word vs PDF.** Bucles `{#}{/}` e inversos `{^}{/}` solo en Word. En PDF cada variable es el nombre de un campo de texto único. Stage 5 valida esto.
7. **Idioma:** todo el trabajo interno y el racional en español. Los tokens son literales del catálogo (no se traducen).

## Manejo de errores

- **Input ambiguo en Stage 1** (no se sabe si es plantilla, qué país, qué tipo de sociedad): pregunta al usuario antes de avanzar.
- **Sin ejemplo diligenciado:** procede igual; Stage 2 mapea por coincidencia de etiqueta + catálogo + plantilla de referencia, y marca más campos como `AMBIGUO` para Stage 4.
- **Actor no disponible en Stage 3:** continúa con los demás y marca explícitamente cuál faltó. No bloquees.
- **PDF no editable (sin campos de formulario, escaneado):** Stage 1 lo detecta; reporta al usuario que para PDF se requiere un PDF con campos de formulario (Adobe Acrobat) o, alternativamente, entregar la configuración como Word. No intentes OCR para inyectar campos.
- **Helper de Word/PDF falla en Stage 5:** entrega el mapa de variables en markdown como fallback y avisa que la inserción final requiere ajuste manual.
- **Usuario no responde Stage 4:** mantén el estado; no expires el pipeline.

## Checklist antes de devolver el entregable final

- [ ] Los 6 stages se ejecutaron en orden (Stage 4 puede ser no-op si no hubo dudas).
- [ ] Si hubo campos sin resolver, el usuario los respondió (no auto-rellenados).
- [ ] El archivo de salida tiene el formato del cliente intacto, solo con tokens insertados.
- [ ] Ningún token usa una variable inexistente (todas verificadas contra el catálogo).
- [ ] Word: bucles/inversos solo donde aplica; PDF: un campo por variable, sin duplicados.
- [ ] El entregable lleva mapa de variables + racional corto.
- [ ] Los campos `SIN_VARIABLE` están listados con su tratamiento (texto fijo / vacío) y motivo.
