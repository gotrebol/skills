---
name: stage-5-configure-deliverable
description: Quinta etapa del pipeline de configuración de plantillas de dictamen. Toma el mapeo consolidado (Stage 3) más las respuestas del usuario (Stage 4) e inserta los tokens de Trébol en el archivo del cliente, preservando exactamente su formato. Según el tipo de archivo invoca al helper de Word (helper-docx-template-filler) o al de PDF (helper-pdf-form-configurator). Produce el archivo configurado listo para subir a la plataforma de Trébol. Úsalo cuando el orquestador lo invoque como quinto paso, o cuando el usuario pida "configurar la plantilla", "insertar las variables" o "generar el archivo final".
---

# Stage 5 — Configure Deliverable (Insertar los tokens)

Quinta etapa. Convierte las decisiones en el archivo configurado real: la misma plantilla del cliente, mismo formato, con los tokens insertados.

## Responsabilidad

**Una sola cosa:** producir el archivo de salida (mismo formato del cliente) con los tokens del mapeo final insertados en los lugares correctos. No redecide mapeos (vienen cerrados de Stage 3/4); solo inserta.

## Input
- `state.template` (estructura y ubicaciones de Stage 1).
- `state.panel_decisions` (mapeo `FINAL` por campo).
- `state.user_answers` (resoluciones de `ASK_USER`/`SIN_VARIABLE`).
- Ruta del archivo original del cliente.

## Proceso

### Paso 1 — Construir el mapeo de inserción definitivo
Funde panel + usuario en una tabla `{field_id, location, token | <vacío/fijo>}`. Resuelve los `ASK_USER` con lo que respondió el usuario. Los `SIN_VARIABLE` quedan como texto fijo o en blanco según se decidió.

### Paso 2 — Elegir helper según formato
- `deliverable_format == docx` → `helpers/helper-docx-template-filler.md`.
- `deliverable_format == pdf_form` → `helpers/helper-pdf-form-configurator.md`.

Pasa al helper: ruta del original, mapeo de inserción, y banderas (Word: qué filas son bucles `{#}{/}`; PDF: cuántas instancias por lista según respondió el usuario).

### Paso 3 — Validaciones tras la inserción (antes de devolver)
- **Formato intacto:** ninguna fuente, color, ancho de columna, encabezado o estilo del cliente cambió. Solo se agregaron tokens.
- **Tokens válidos:** cada token insertado existe en el grounding y está bien escrito (sin espacios dentro de llaves, sin mayúsculas accidentales, llaves balanceadas).
- **Word:** bucles `{#}{/}` e inversos `{^}{/}` solo donde corresponde; las variables dentro de un bucle son relativas.
- **PDF:** cada token está insertado como valor de un campo de texto **único** (o como nombre del campo si la plantilla usa Convención B); no hay tokens duplicados en dos campos distintos; no se usaron bucles.
- **Cobertura:** todo campo `FINAL` tiene su token insertado; todo `SIN_VARIABLE` quedó como se decidió. Nada quedó a medias.

Si una validación falla, intenta corregir una vez; si persiste, reporta a Stage 6 con `formatting_issues`.

### Paso 4 — Verificación de tokens en Word (recomendado)
Tras empacar el `.docx`, re-extrae el texto y confirma que cada token aparece **completo en un solo fragmento** (Word a veces parte el token en varios runs, lo que impediría el reemplazo en Trébol). Si un token quedó partido, normaliza el formato de ese token.

## Output (a Stage 6)
```
{
  "success": true,
  "output_path": ".../{nombre}_configurada.{docx|pdf}",
  "format": "docx|pdf_form",
  "tokens_inserted": N,
  "fields_fixed_text": [F033, ...],     # SIN_VARIABLE como texto fijo/blanco
  "validation": { "format_intact": true, "tokens_valid": true, "loops_ok": true },
  "formatting_issues": []
}
```

## Reglas duras
1. **Formato del cliente sagrado.** Mismo archivo, solo tokens añadidos. No brand guidelines de Trébol.
2. **No redecidas mapeos.** Vienen cerrados; si algo llega ambiguo, detente y devuelve a Stage 3/4, no improvises.
3. **No inventes variables** al insertar.
4. **Trabaja sobre copia** del original (nunca sobre uploads).
5. **Word ≠ PDF.** Aplica las reglas de cada formato (bucles solo Word; un campo por variable en PDF).
6. **Token completo en un solo run** (Word) — verificar.

## Manejo de errores
- **Helper falla:** entrega el mapa de variables en markdown como fallback y reporta a Stage 6 que la inserción requiere ajuste manual.
- **PDF con campo cuyo nombre no se puede renombrar:** reporta el campo; no lo dejes con nombre incorrecto.
- **Token partido irreparable en Word:** reporta el campo en `formatting_issues` para revisión manual; no lo dejes silenciosamente roto.

## Checklist
- [ ] Mapeo de inserción definitivo construido (panel + usuario).
- [ ] Helper correcto según formato.
- [ ] Formato del cliente intacto.
- [ ] Tokens válidos y bien escritos; Word: completos en un run; PDF: campos únicos.
- [ ] Cobertura total (FINAL insertado, SIN_VARIABLE resuelto).
- [ ] Output estructurado a Stage 6, con issues reportados si los hubo.
