---
name: stage-6-rationale-packaging
description: Sexta y última etapa del pipeline de configuración de plantillas de dictamen. Toma el archivo configurado por Stage 5 y lo empaqueta para entrega, adjuntando un mapa de variables auditable (campo del cliente → variable de Trébol), un racional corto (3-6 bullets) de las decisiones clave, y la lista de campos sin variable con su tratamiento. Separa el archivo limpio que se sube a Trébol del documento de trazabilidad interno. Úsalo cuando el orquestador lo invoque como sexto paso, o cuando el usuario pida "empaquetar", "agregar el mapa de variables" o "preparar la entrega final".
---

# Stage 6 — Rationale Packaging (Entrega + trazabilidad)

Sexta etapa. Garantiza que la plantilla configurada se entregue con su mapa de variables y el racional de las decisiones, para auditoría y para que cualquiera del equipo entienda y mantenga la configuración después.

## Responsabilidad

**Una sola cosa:** empaquetar el entregable. El archivo configurado se entrega limpio (es el que se sube a Trébol); aparte se entrega un documento de trazabilidad con el mapa de variables y el racional.

## Input
- `state.configured_template` (de Stage 5: ruta + métricas).
- `state.panel_decisions` (decisiones, disensos, notas).
- `state.user_answers`.

## Proceso

### Paso 1 — Mapa de variables (auditable)
Tabla, una fila por campo del cliente:

| Campo (etiqueta) | Ubicación | Variable de Trébol | Forma | Confianza | Nota |
|---|---|---|---|---|---|

Incluye TODOS los campos, también los `SIN_VARIABLE` (con "—" en variable y la nota de tratamiento).

### Paso 2 — Racional corto (3–6 bullets)
Solo las decisiones que importan: ambigüedades resueltas, gotchas aplicados (numeración, delegate, separador Colombia), campos sin variable y por qué, y cualquier precedente que marcó el CPO. No narres todo el pipeline.

### Paso 3 — Pendientes de verificación
Lista los tokens marcados como "verificar contra una verificación real" (ej. `delegate`, variantes no documentadas, separador Colombia). Estos son candidatos a `corrections/` una vez confirmados, y deben probarse exportando una verificación de prueba antes de dar la plantilla por cerrada.

### Paso 4 — Cómo probar (recordatorio operativo)
Breve nota para el usuario: subir la plantilla en Ajustes → "Plantillas de formatos", correr el endpoint de exportación contra una verificación finalizada de prueba, y revisar que los IDs se reemplacen. Si algún ID queda literal, revisar llaves/espacios/numeración (ver `grounding/sintaxis-plantillas.md` sección 7).

### Paso 5 — Formato del documento de trazabilidad
Markdown por defecto (rápido y editable). Si el usuario pide Word, genera un `.docx` Trébol-branded (este documento es interno, no es la plantilla del cliente, así que aquí **sí** aplican los brand guidelines de `trebol-brand-guidelines`). El documento de trazabilidad debe llevar visible "DOCUMENTO INTERNO — TRAZABILIDAD DE CONFIGURACIÓN".

### Paso 6 — Entregar
Usa `present_files` con, en este orden:
1. El archivo configurado (la plantilla lista para subir a Trébol) — es lo más relevante.
2. El documento de mapa de variables + racional.

## Output al usuario (estructura)
```
# Plantilla configurada — {client_name}

[present_files: plantilla_configurada.docx/pdf, mapa_variables_y_racional.md]

## Racional (resumen)
- ...
- ...

## Pendientes de verificación
- ...
```

## Reglas duras
1. **Dos artefactos separados:** plantilla limpia (a Trébol) y trazabilidad (interna). No mezclar el racional dentro de la plantilla del cliente.
2. **Mapa completo:** todos los campos, incluidos los sin variable.
3. **Racional corto y útil**, no un relato del proceso.
4. **Pendientes de verificación explícitos**, con recordatorio de probar contra una verificación real.
5. **Brand guidelines solo en el documento interno** si se genera en Word, nunca en la plantilla del cliente.

## Checklist
- [ ] Archivo configurado entregado limpio (sin anotaciones internas).
- [ ] Mapa de variables completo (incluye SIN_VARIABLE).
- [ ] Racional de 3–6 bullets con las decisiones clave.
- [ ] Pendientes de verificación listados.
- [ ] Recordatorio de cómo subir y probar la plantilla.
- [ ] `present_files` con la plantilla primero.
