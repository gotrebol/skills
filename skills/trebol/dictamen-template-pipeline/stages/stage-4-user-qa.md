---
name: stage-4-user-qa
description: Cuarta etapa del pipeline de configuración de plantillas de dictamen. Produce un markdown breve y escaneable con solo los campos que el panel (Stage 3) no pudo resolver (campos sin variable clara, ambigüedades, decisiones de negocio). Es el único punto de pausa humana del pipeline — se detiene aquí y espera respuestas reales antes de avanzar a Stage 5. Si no quedaron campos abiertos, devuelve "sin dudas" y no pausa. Úsalo cuando el orquestador lo invoque como cuarto paso, o cuando el usuario pida "qué necesitas confirmar antes de configurar la plantilla".
---

# Stage 4 — User Q&A (Punto de pausa humana)

Cuarta etapa. Reúne las dudas abiertas tras la discusión del panel y las presenta al usuario de forma corta y accionable. Único checkpoint humano del pipeline.

## Responsabilidad

**Una sola cosa:** listar los campos `ASK_USER` (y los `SIN_VARIABLE` cuyo tratamiento conviene confirmar) en un markdown breve que el usuario responde rápido, idealmente desde el móvil. No configura, no decide por el usuario.

## Cuándo NO pausa
Si Stage 3 no dejó ningún `ASK_USER` y los `SIN_VARIABLE` son obvios (firmas, sellos, notas internas), devuelve:

```
# Dudas para el usuario
Sin dudas abiertas — el panel resolvió todos los campos. Avanzando a la configuración.
```

…y el orquestador salta directo a Stage 5.

## Formato del markdown (cuando sí hay dudas)

Corto, escaneable, una pregunta por bloque, opción por defecto sugerida cuando sea posible:

```
# Confirmaciones antes de configurar la plantilla — {client_name}

Necesito tu ok en {N} puntos. Responde con el número y tu elección.

1. **"TIPO DE GARANTÍA OTORGADA"** (sección Datos Generales)
   ¿Lo lleno con la capacidad estatutaria que extrae Trébol (`{legal_businessAssetPledging}`, sale "sí/no"), o es un dato que captura tu equipo a mano?
   - a) Variable de Trébol  ·  b) Campo manual (lo dejo en blanco)
   _Sugerencia: (a) si el campo solo refleja el objeto social._

2. **"OBSERVACIONES DEL ANALISTA"** (final del dictamen)
   No hay variable de Trébol para esto. ¿Lo dejo en blanco para llenado manual, o lo quito?
   - a) Dejar en blanco  ·  b) Quitar la sección

3. **Apoderados (PDF)**: tu plantilla es PDF, no soporta listas dinámicas. ¿Para cuántos apoderados dejo campos? (Trébol llenará los que existan)
   - a) 2  ·  b) 3  ·  c) 4
```

## Reglas duras
1. **No avances sin respuestas.** Si el orquestador no tiene `state.user_answers`, el pipeline se queda aquí.
2. **No auto-respondas.** Aunque "sepas" la respuesta probable, el usuario decide los campos de negocio y los sin-variable.
3. **Solo lo no resuelto.** No re-preguntes campos ya decididos `FINAL` por el panel. El usuario no debe revisar todo el mapeo, solo lo dudoso.
4. **Cerrado y corto.** Preguntas con opciones, no ensayos. Máximo una decisión por punto.
5. **Respuesta parcial:** procesa lo respondido, vuelve a preguntar solo lo que falta. No avances con huecos.

## Output
Cuando el usuario responde, estructura `state.user_answers` como `{field_id: decisión}` y devuélvelo al orquestador para Stage 5. Marca claramente cualquier punto que siga sin respuesta.

## Checklist
- [ ] Solo aparecen campos no resueltos por el panel.
- [ ] Cada punto es una pregunta cerrada con opciones.
- [ ] Hay sugerencia por defecto donde aplica.
- [ ] El markdown es corto y escaneable.
- [ ] No se auto-respondió nada; el pipeline pausa hasta recibir respuestas.
