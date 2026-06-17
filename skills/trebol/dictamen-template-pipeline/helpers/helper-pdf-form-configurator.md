---
name: helper-pdf-form-configurator
description: Helper del pipeline de configuración de plantillas de dictamen para configurar plantillas en PDF editable (PDF con campos de formulario / AcroForm) de clientes de Trébol. La convención confirmada en producción (Convención A) es que el token {variable} va como VALOR del campo de formulario y el nombre del campo es arbitrario; si la plantilla ya usa nombre=variable (Convención B), se respeta. Un token por campo, sin duplicados, sin bucles ni inversos (esos solo funcionan en Word). Las listas se expanden como instancias fijas indexadas. Úsalo cuando Stage 5 lo invoque para producir el entregable en PDF. NO decide qué variable va dónde (viene cerrado de Stage 3/4) — recibe un mapeo de inserción ya resuelto y configura los campos preservando el diseño visual del PDF del cliente.
---

# Helper — Configurador de Plantillas PDF (formulario / AcroForm)

Configuras la plantilla PDF editable del cliente insertando los tokens de Trébol en los **valores de sus campos de formulario**, **sin tocar el diseño visual**. El cliente espera su PDF idéntico a la vista, solo que sus campos ahora tienen el token como valor por defecto y la plataforma los podrá autollenar al exportar (Convención A, confirmada en producción).

**No redactas ni decides mapeos.** Recibes de Stage 5 un mapeo de inserción cerrado (`{field_id, location, variable | texto_fijo | vacío}`) y las instancias por lista que el usuario confirmó en Stage 4. Tu trabajo es ejecución técnica fiel.

## La regla central del PDF (interiorizar antes de empezar)

En PDF **no hay motor de plantillas con bucles**. El reemplazo se hace campo por campo de formulario. Hay **dos convenciones** y debes **replicar la que ya trae la plantilla del cliente**:

- **Convención A — token en el VALOR del campo (confirmada en producción, la más común):** el token `{variable}` se escribe como **valor / texto por defecto** del campo de formulario; el **nombre del campo es arbitrario** (`Texto100`, etc.) y se deja como está. Ejemplo real documentado en `grounding/example-client-mapping.md`. Si el cliente entrega un PDF con campos sin tokens, usa esta: pon el token como valor de cada campo.
- **Convención B — nombre del campo = la variable:** algunos PDFs nombran el campo exactamente como `{legal_businessName}`. Si la plantilla ya viene así, respétala.

Reglas comunes a ambas convenciones:
- **Un token único por campo.** No reutilices el mismo token en dos campos que deban llenarse distinto. Nunca dupliques tokens que rompan el llenado.
- **No hay bucles `{#}{/}` ni inversos `{^}{/}`** en PDF. Si el mapeo trae un bucle (viene de un caso Word), tradúcelo a instancias indexadas.
- **Las listas se expanden a campos por instancia.** Para N apoderados: `{key_person_1_...}`, `{key_person_2_...}`, … (apoderados 1-based). Para N accionistas: `{shareholder_0_...}`, … (0-based). El número de instancias lo confirmó el usuario en Stage 4; no lo inventes.

## Antes de tocar nada

1. **Lee `/mnt/skills/public/pdf/SKILL.md`** primero. Es obligatorio: define cómo manipular PDF y formularios en este entorno. No escribas código antes de leerlo.
2. **Trabaja sobre una copia.** Nunca edites el archivo en `/mnt/user-data/uploads`.
3. **Confirma que el PDF tiene campos de formulario.** Si es un PDF plano (escaneado, sin AcroForm), no se puede configurar por nombre de campo: repórtalo a Stage 5 (la salida correcta es pedir un PDF con campos de formulario hecho en Acrobat "Preparar formulario", o entregar en Word). **No intentes OCR ni inyectar campos donde el cliente no los puso** — alterarías el diseño.

## Cómo configurar cada campo

Aplica **siempre Convención A** salvo que la plantilla del cliente ya use Convención B (ver sección introductoria). En Convención A el nombre del campo se deja como está y el token se escribe como **valor / texto por defecto** del campo.

### Campo simple — Convención A (por defecto)
Localiza el campo de formulario correspondiente. Escribe el token como **valor del campo**: `{tax_businessTaxId}`. El nombre del campo (`Texto100`, `Field_23`, etc.) queda intacto. Sin espacios dentro de las llaves, minúsculas correctas, llaves balanceadas. Conserva posición, tamaño, fuente y propiedades visuales.

Si la plantilla ya usa Convención B (nombre del campo = variable), respeta esa convención: en ese caso el nombre del campo pasa a ser `{tax_businessTaxId}` y el valor queda vacío.

### Listas (apoderados, accionistas, consejo)
- Expande a tantas instancias como confirmó el usuario, con la **numeración correcta por familia** (apoderados 1-based; accionistas/consejo/administradores/comisarios 0-based).
- Si el PDF del cliente ya trae campos para N instancias, escribe el token como valor de cada campo en orden. Si trae menos campos que instancias necesarias, repórtalo (no agregues campos que descuadren el diseño sin confirmar con el usuario).
- Cada subcampo de cada instancia es su propio campo único. Ejemplo en Convención A: campo con nombre arbitrario, valor `{key_person_1_name}`; siguiente campo, valor `{key_person_1_role}`; siguiente, valor `{key_person_2_name}`…

### Casillas / poderes (x_mark)
- Si el diseño usa casillas o campos de marcación, escribe el token `*_x_mark` como **valor** del campo (Convención A). Si la plantilla ya nombra el campo como la variable, respeta Convención B.
- Una casilla por poder; respeta numeración 1-based de apoderados.

### FUENTE y condicionales
- En PDF no hay inverso ni bucles. La FUENTE se configura como campo(s) de texto con el token de la variable de fuente como valor; si el dato no existe, el campo queda vacío al llenarse. Si el diseño del cliente requiere un texto alternativo de "no encontrado", es captura/diseño del cliente — déjalo como esté salvo que el mapeo indique otra cosa.

## Preservación del diseño (sagrado)

No muevas, redimensiones ni reestilices ningún campo o elemento visual. No cambies fuentes, colores, posiciones ni el contenido estático del PDF. **No apliques brand guidelines de Trébol.** Lo único que cambia es el **valor por defecto** de los campos (Convención A) o su nombre (Convención B si la plantilla ya la usa).

## Campos SIN_VARIABLE
- **Texto fijo:** si se decidió texto fijo, escríbelo como valor por defecto del campo (Convención A) o déjalo como contenido estático, según el diseño.
- **En blanco:** deja el campo con valor vacío (no pongas un token inventado).

## Output (a Stage 5)
```
{
  "success": true,
  "output_path": ".../{nombre}_configurada.pdf",
  "fields_named": N,
  "list_instances": [{ "list": "key_person", "base": 1, "count": 4 }, { "list": "shareholder", "base": 0, "count": 3 }],
  "duplicate_names_found": [],       # debe quedar vacío; cualquier duplicado es fallo
  "no_formfields_warning": false,    # true si el PDF no tenía AcroForm
  "design_intact": true
}
```

## Reglas duras
1. **Lee el SKILL.md público de pdf antes de codear.**
2. **Copia primero; nunca edites el upload.**
3. **Un campo único por variable — cero duplicados de nombre.**
4. **Sin bucles ni inversos; listas → instancias indexadas** con numeración correcta por familia.
5. **Diseño del cliente intacto; sin brand guidelines de Trébol.** Solo cambian nombres de campo.
6. **No inventes variables ni redecidas mapeos.**
7. **Si no hay campos de formulario, repórtalo** — no inyectes campos ni hagas OCR.

## Manejo de errores
- **PDF sin AcroForm:** `no_formfields_warning: true` y reporta a Stage 5 que se requiere PDF con campos de formulario o entrega en Word. No improvises.
- **Nombre de campo que no se puede renombrar** (campo protegido/firmado): reporta el campo; no lo dejes con nombre incorrecto.
- **Menos campos en el PDF que instancias requeridas en una lista:** reporta el desajuste a Stage 5; no agregues campos que rompan el diseño sin confirmación.
- **Mapeo ambiguo:** devuelve a Stage 5 indicando el campo no resuelto; no improvises.

## Checklist
- [ ] Leí `/mnt/skills/public/pdf/SKILL.md`.
- [ ] Trabajé sobre copia, no sobre el upload.
- [ ] El PDF tenía campos de formulario (o reporté que no).
- [ ] Cada variable es el nombre de un campo único; cero duplicados.
- [ ] Listas expandidas a instancias con numeración correcta (1-based apoderados / 0-based el resto).
- [ ] Sin bucles ni inversos.
- [ ] Diseño del cliente intacto; sin brand guidelines de Trébol.
- [ ] SIN_VARIABLE tratado como se decidió.
- [ ] Output estructurado a Stage 5 con issues reportados.
