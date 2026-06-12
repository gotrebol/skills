---
name: stage-3-panel-discussion
description: Tercera etapa del pipeline de configuración de plantillas de dictamen. Convoca a los 3 actores de Trébol (CPO, ingeniero experto en variables de plantillas, y persona legal de dictámenes) para revisar el mapeo candidato de Stage 2 y discutir qué va en cada campo, resolviendo desacuerdos y produciendo un mapeo consolidado con decisión por campo (variable final / sin variable / preguntar al usuario), disensos registrados y la lista de campos que requieren input humano. Úsalo cuando el orquestador lo invoque como tercer paso, o cuando el usuario pida "revisar el mapeo", "discutir con el equipo" o "validar las variables".
---

# Stage 3 — Panel Discussion (Los actores discuten qué llenar)

Tercera etapa. Los 3 actores revisan el mapeo candidato y deciden, campo por campo, qué token va (o si no va, o si hay que preguntar al usuario).

## Responsabilidad

**Una sola cosa:** convertir el mapeo candidato de Stage 2 en un **mapeo consolidado con decisiones**, resolviendo desacuerdos entre actores y dejando registro de disensos. No edita el archivo (eso es Stage 5).

## Actores (todos reviewers; ninguno redactó el borrador, para evitar autocomplacencia)

Adopta las tres perspectivas leyendo cada `.md` bajo `personas/` y razonando desde ese rol. Produce el dictamen de cada una de forma **independiente** primero, luego concilia.

1. `persona-trebol-cpo-reviewer` — producto, experiencia del cliente, precedente, escalabilidad y mantenibilidad de la plantilla.
2. `persona-trebol-template-engineer` — corrección técnica de cada variable: que exista, numeración correcta, bucle vs PDF, fuentes/inversos, gotchas (delegate, x_mark, separadores Colombia, espacios/mayúsculas en llaves).
3. `persona-trebol-legal-dictamen` — corrección jurídica del mapeo: que cada variable corresponda al concepto legal correcto del dictamen (no confundir poder de dominio con administración, que la FUENTE cite folio/RPPC adecuadamente, suficiencia regulatoria).

## Proceso

### Paso 1 — Dictamen independiente
Pasa `state.template` + `state.candidate_mapping` + grounding a cada actor. Cada uno devuelve, por campo: `agree` / `change` (con la variable corregida) / `flag` (preocupación) / `ask_user` (requiere humano), con racional.

### Paso 2 — Conciliación por campo
Para cada campo, junta los 3 dictámenes y decide:
- **Consenso** (los 3 coinciden) → `decision = variable acordada`.
- **Desacuerdo técnico** (el ingeniero corrige la variable) → prevalece el ingeniero en lo técnico (qué variable existe y cómo se escribe), pero el legal puede vetar si la variable correcta técnicamente no corresponde al concepto legal del campo (entonces `ask_user` o `SIN_VARIABLE`).
- **Desacuerdo legal** (la variable existe pero el legal dice que no corresponde al concepto) → prevalece el criterio legal sobre el "encaja técnicamente"; registrar disenso del ingeniero si lo hay.
- **Preocupación de producto/precedente** (CPO) → no bloquea una variable correcta, pero se registra como nota para el racional (ej. "este campo crea precedente: futuros clientes esperarán lo mismo").
- **No se resuelve internamente** (ambigüedad real, decisión de negocio, campo sin variable cuya intención no está clara) → `decision = ask_user` con la pregunta concreta para Stage 4.

### Paso 3 — Decisión por campo
Cada campo queda con uno de:
- `FINAL` + token(s) definitivos.
- `SIN_VARIABLE` + tratamiento sugerido (texto fijo / dejar en blanco) + motivo.
- `ASK_USER` + pregunta cerrada para Stage 4.

### Paso 4 — Registrar disensos y notas
Cuando un actor quede en desacuerdo con la decisión, registra `dissent` con su nombre y razón. Las preocupaciones de precedente/mantenibilidad del CPO van a `notes` (alimentan el racional de Stage 6).

## Output (markdown para Stage 4/5)

```
# Panel Decisions
## Resumen
- total_fields: N
- FINAL: n1
- SIN_VARIABLE: n2
- ASK_USER: n3

## F014  "FACULTADES DEL APODERADO"
- decision: FINAL
- variable: {key_person_1_power_administration_x_mark} ... (6 poderes, 1-based)
- dissent: []
- notes: "Ingeniero: 'delegate' no está en doc web; se mantiene porque la referencia lo usa, pendiente verificar contra verificación real (candidato a corrections)."
- rationale: "Legal confirma que las 6 facultades corresponden al concepto de poderes del dictamen; ingeniero confirma sintaxis x_mark y numeración 1-based."

## F051  "TIPO DE GARANTÍA OTORGADA"
- decision: ASK_USER
- question: "¿Este campo debe llenarse con la capacidad estatutaria {legal_businessAssetPledging} (sí/no que extrae Trébol) o es un dato operativo que captura el cliente manualmente?"
- rationale: "El ingeniero ve una variable cercana; el legal duda que corresponda al mismo concepto. Decisión de negocio → usuario."
```

## Reglas duras
1. **El ingeniero manda en lo técnico** (existencia y escritura de la variable); **el legal manda en la correspondencia conceptual** (que la variable signifique lo que el campo del dictamen pide). Si chocan, gana el legal en "qué concepto" y el ingeniero en "cómo se escribe la variable de ese concepto".
2. **No inventar variables** en la conciliación, igual que Stage 2.
3. **Todo desacuerdo no resuelto → `ASK_USER`,** no un mapeo a la fuerza.
4. **Racional y disensos siempre registrados.** Invariante del pipeline.
5. **No edites el archivo.** Solo decides. Stage 5 inserta.
6. **Si un actor no está disponible,** continúa con los otros dos y marca explícitamente cuál faltó en `notes`.

## Manejo de errores
- **Variable que ningún actor logra confirmar que existe:** `SIN_VARIABLE` o `ASK_USER` (nunca FINAL con una variable dudosa). Sugiere verificar contra la doc web / una verificación real.
- **El cliente ya tokenizó mal un campo:** decisión `FINAL` con el token corregido + `notes` explicando la corrección.
- **Plantilla con secciones que Trébol no soporta** (consulta externa nueva): `SIN_VARIABLE` con nota; candidato a futura variable.

## Checklist
- [ ] Los 3 actores dieron dictamen (o se marcó cuál faltó).
- [ ] Cada campo tiene decisión `FINAL`/`SIN_VARIABLE`/`ASK_USER`.
- [ ] Desacuerdos registrados como `dissent`.
- [ ] Preocupaciones de precedente/mantenibilidad en `notes`.
- [ ] Ninguna decisión `FINAL` usa una variable no confirmada.
- [ ] Preguntas de `ASK_USER` formuladas cerradas para Stage 4.
