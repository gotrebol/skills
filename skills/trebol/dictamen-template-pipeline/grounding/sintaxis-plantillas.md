# Sintaxis del Motor de Plantillas de Trébol (grounding)

Cómo se escriben los tokens que Trébol reemplaza al exportar. Aplica igual a Word y PDF salvo donde se indica.

## 1. Variable simple

```
{legal_businessName}
```

Trébol la reemplaza por el valor extraído. Si la variable no tiene dato, queda vacía (o, en algunos casos, con un fallback definido por un inverso — ver abajo).

**Reglas de sintaxis (causan que el ID no se reemplace si se violan):**
- Las llaves `{` `}` son obligatorias y deben envolver exactamente el ID.
- **Sin espacios dentro de las llaves.** `{ key_person_1_source_notaryState}` (con espacio inicial) NO se reemplaza. (Esta errata aparece en la propia plantilla de referencia — no copiarla.)
- El ID debe existir en el diccionario (`variable-catalog.md`) tal cual, respetando mayúsculas/minúsculas y guiones bajos. `{KEY_PERSON_2_ROLE_NAME}` en mayúsculas probablemente NO funcione; usar `{key_person_2_role_name}`.
- En Word, evitar que el editor parta el token en varios "runs" (negritas parciales, autocorrección). Si una llave queda con formato distinto al ID, el motor puede no detectarla. Escribir el token completo de un tirón y, si hace falta, limpiar el formato del token.

## 2. Bucle / sección (listas)  — **solo Word, NO PDF**

Repite el bloque interior por cada elemento de la lista:

```
{#shareholders}{name} — {share_percentage}%{/}
```

- Apertura `{#nombre}`, cierre `{/}` o `{/nombre}`. Dentro del bloque, las variables son **relativas** al elemento (`{name}`, no `{shareholders_0_name}`).
- Listas conocidas: `shareholders`, `board_members`, `executives`, `auditors`, `key_person_1_powers`, `tax_businessMainActivity`, `external_sources_renapo`, `external_sources_siger`, `external_sources_ine_validation`, `external_sources_fiel_sello`, `external_sources_aml_validation` (+ sub-bloques `aml_*`), `ubos_business_0_shareholders`, `ubos_business_0_ubos_person_0_name`, `tax_additionalData`/`data`, `documents`, etc. (ver catálogo).
- **En PDF los bucles no funcionan.** Para PDF se configura un campo por cada instancia conocida (ej. `key_person_1`, `key_person_2`) o se acota el alcance — ver `helper-pdf-form-configurator`.

## 3. Sección condicional / inversa

- **Condicional (`#`)**: renderiza el contenido si el valor es verdadero / la lista tiene elementos.
  ```
  {#registration_hasConstitutive}Escritura pública número {registration_actaNumber}…{/}
  ```
- **Inversa (`^`)**: renderiza el contenido si el valor es falso / la lista está vacía. Se usa para los textos de "faltó el documento":
  ```
  {^registration_hasConstitutive}Faltó subir acta constitutiva{/}
  ```
- Patrón típico de **FUENTE** (provenance + fallback), muy común en dictámenes:
  ```
  {#key_person_1_source_number}Escritura pública número {key_person_1_source_number} de fecha {key_person_1_source_date}, otorgada ante la fe del Licenciado(a) {key_person_1_source_notaryName}…{/}{^key_person_1_source_number}No se encontraron poderes para el firmante #1{/}
  ```

## 4. Marcas tipo "X" (`x_mark`)

Variables que renderizan `"X"` cuando una condición es verdadera, para checkboxes/casillas:
- Género: `{is_man_x_mark}`, `{is_woman_x_mark}` (dentro de bucle) o `{board_member_0_is_man_x_mark}` (individual).
- Facultades: `{key_person_1_power_administration_x_mark}` (marca X si cuenta con el poder).
Útiles para plantillas con tablas de "✔ / casilla" en lugar de texto.

## 5. Numeración (recordatorio crítico)

- Apoderados: **1-based** → `key_person_1`, `key_person_2`.
- Accionistas / consejo / administradores / comisarios: **0-based** → `shareholder_0`, `board_member_0`, `executive_0`, `auditor_0`.
- Colombia / UBOs: `{n}` documento, `{i}` primer nivel, `{j}` segundo nivel.
- UBOs MX (referencia): `ubos_business_0`, `ubos_person_0`.

## 6. Diferencias Word vs PDF (resumen para Stage 5)

| Capacidad | Word (.docx) | PDF editable |
|---|---|---|
| Variable simple `{var}` | Sí, como texto inline | Sí, como **valor del campo de texto** (nombre del campo es arbitrario) |
| Bucles `{#}{/}` | Sí | **No** |
| Inversos `{^}{/}` | Sí | **No** (se resuelve dejando el campo vacío) |
| `x_mark` | Sí | Sí (campo de texto que recibirá "X") |
| Instancias repetidas | Bucle | Un campo por instancia conocida |

**PDF — reglas de Adobe Acrobat (doc /plantillas/configuracion):**
- Usar siempre **campos de texto** (no checkbox/radio).
- **Un campo nuevo por variable.** Nunca copiar/pegar el mismo campo (campos con el mismo nombre rompen la exportación).
- **Convención A (confirmada en producción, la más común):** escribe el token `{variable}` como **valor / texto por defecto** del campo; el nombre del campo es arbitrario y se deja como está.
- **Convención B:** algunos PDFs nombran el campo como `{legal_businessName}`. Si la plantilla ya viene así, respétala.
- Replicar siempre la convención que ya trae la plantilla del cliente; ver `grounding/example-client-mapping.md` para un ejemplo real de Convención A.

## 7. Errores comunes (de la doc /plantillas/gestion "Solución de problemas")

- IDs no se reemplazan → revisar llaves, espacios extra, y que el ID exista en la doc.
- Verificar numeración (`-1`, `-2`, índices `-0`, `-1`) y que la verificación tenga items del tipo correcto en estado "finished".
- Archivo no carga → debe ser PDF o .docx, no corrupto, sin contraseña.
