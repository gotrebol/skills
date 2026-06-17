---
name: corrections
description: Correcciones conocidas al pipeline de configuración de plantillas de dictamen. Registra errores que el pipeline cometió al mapear o escribir variables de Trébol en plantillas pasadas, y los mapeos/escrituras correctos que deben usarse. Este skill tiene MÁXIMA PRIORIDAD sobre cualquier otro grounding — ante cualquier conflicto entre una entrada aquí y el catálogo de variables, la documentación web de gotrebol.com o la plantilla de referencia, prevalece este archivo. El orquestador lo carga automáticamente en el Paso 0 antes de arrancar el pipeline.
---

# Correcciones Conocidas — Pipeline de Configuración de Plantillas de Dictamen

Este archivo tiene **máxima prioridad** sobre cualquier otro documento de grounding. Ante conflicto entre una entrada aquí y `grounding/variable-catalog.md`, `grounding/sintaxis-plantillas.md`, `grounding/example-template-reference.md`, o la documentación viva en `gotrebol.com/docs/plantillas/`, **prevalece este archivo**.

Su razón de ser: la documentación y la plantilla de referencia tienen erratas y huecos. Cuando una variable o sintaxis se confirma (o se descarta) **contra una verificación real en la plataforma**, la verdad operativa vive aquí, no en la doc.

**Responsable:** content manager del pipeline (mismo dueño que en el pipeline de cuestionarios).
**Contribuidores:** cualquiera del equipo puede proponer entradas; se validan antes de agregar.

---

## Cómo agregar una corrección

Cuando se detecta que el pipeline mapeó/escribió mal una variable y se confirma lo correcto contra una verificación real:

1. Identificar qué tipo de campo o variable lo activa.
2. Documentar qué hacía mal el pipeline (variable inexistente, mal escrita, numeración equivocada, bucle donde no va, etc.).
3. Documentar lo correcto, con el token exacto y la fuente (quién/qué lo confirmó).
4. Agregar la entrada al final de este archivo, re-empaquetar el zip y re-subir la skill a Claude.ai.

**Formato de cada entrada:**

```
## Corrección: [tema en 3-5 palabras]
**Campo / variable tipo:** [campo del dictamen o variable que la activa; keywords]
**Error conocido:** [qué hacía mal el pipeline — token exacto incorrecto]
**Correcto:** [el token/sintaxis correcto exacto]
**Fuente:** [verificación real / ingeniero / doc actualizada / quién confirmó]
**Fecha:** YYYY-MM-DD
```

---

## Candidatos a verificar (gotchas conocidos, AÚN NO confirmados)

Estos NO son correcciones activas todavía — son puntos dudosos detectados al destilar el grounding. El pipeline debe **marcarlos para verificación** cuando aparezcan, y solo se vuelven corrección activa (arriba, con prioridad máxima) una vez confirmados contra una verificación real. Hasta entonces, se sigue la plantilla de referencia pero se deja `flag`.

- **Poder `delegate`:** **CONFIRMADO real** (ver corrección activa abajo). Ya no es candidato.
- **Separador en Colombia (`_` vs `-`):** la doc de Colombia mezcla guion bajo y guion medio en los nombres de variable. Confirmar cuál acepta la cuenta del cliente antes de cerrar una plantilla colombiana; no mezclar ambos en un mismo documento. (El payload de ejemplo `dictionary-example.json` es de México y usa solo `_`, así que no resuelve la duda de Colombia.)
- **Inverso plural `board_members_0_names`:** la referencia usa el inverso en plural (`{^board_members_0_names}`) frente a la variable individual `board_member_0`. Confirmar la grafía exacta del inverso para el consejo.
- **Numeración por familia:** recordatorio permanente — apoderados son **1-based** (`key_person_1`), pero accionistas, consejo, administradores y comisarios son **0-based** (`shareholder_0`, `board_member_0`, `executive_0`, `auditor_0`). No es un error a corregir sino la regla; se lista aquí por ser la confusión más cara.

---

## Correcciones activas

## Corrección: poder `delegate` es real
**Campo / variable tipo:** facultades/poderes de apoderado; familia `key_person_N_power_delegate_*`.
**Error conocido:** se dudaba si `delegate` existía (la doc web solo listaba 5 poderes).
**Correcto:** `delegate` es un poder real (otorgar/delegar/sustituir poderes), con sub-campos `_x_mark`, `_signature_type`, `_duration`, `_limits_description`. La etiqueta de cliente "PODER OTORGAR/DELEGAR/SUSTITUIR PODERES" mapea a `delegate`. La lista completa de poderes es 6: administration, assets_management, loans, bank_accounts, lawsuits_and_collections, delegate.
**Fuente:** template real de cliente (PDF AcroForm, ver `grounding/example-client-mapping.md`).
**Fecha:** 2026-06-11

## Corrección: en PDF el token va en el VALOR del campo (no en el nombre)
**Campo / variable tipo:** configuración de plantillas en PDF de formulario (AcroForm).
**Error conocido:** el helper de PDF asumía que "el nombre del campo = la variable". En una plantilla real configurada por Trébol los nombres de campo son arbitrarios (`Texto100`, etc.) y el token `{variable}` está puesto como **valor / texto por defecto** del campo.
**Correcto:** al configurar PDF, escribe el token `{variable}` como **valor del campo de formulario**; deja el nombre del campo como venga. Replica la convención de la plantilla del cliente. Un token único por campo; sin duplicados; sin bucles. (Si una plantilla del cliente sí usa nombre=variable, respétala; pero la convención confirmada en producción es token-en-valor.)
**Fuente:** template real de cliente (PDF AcroForm ~1.092 campos); ver `grounding/example-client-mapping.md`.
**Fecha:** 2026-06-11

## Corrección: grafía de `tax_additionalData`
**Campo / variable tipo:** datos fiscales adicionales; cualquier variable de la familia `tax_additionalData*`.
**Error conocido:** la plantilla de referencia (`example-template-reference.md`) escribe la variable como `tax_additonalData` (sin la "i": addi**t**onal), una errata. Replicarla produce un token que no se reemplaza.
**Correcto:** `tax_additionalData` (con la "i": addi**ti**onal). Confirmado contra el payload real `dictionary-example.json`, que trae las claves reales `tax_additionalData`, `tax_additionalData_source_itemId`, `tax_additionalData_source_itemType`.
**Fuente:** payload real `grounding/dictionary-example.json` (verificación real de cliente).
**Fecha:** 2026-06-04

---

_(Las demás entradas confirmadas contra verificaciones reales se agregan aquí y tienen prioridad máxima sobre todo el grounding.)_
