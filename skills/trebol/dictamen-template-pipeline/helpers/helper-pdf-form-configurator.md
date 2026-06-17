---
name: helper-pdf-form-configurator
description: Helper del pipeline de configuración de plantillas de dictamen para configurar plantillas en PDF editable (PDF con campos de formulario / AcroForm) de clientes de Trébol. La regla central del PDF es que cada variable de Trébol se configura como el NOMBRE de un campo de formulario (ej. un campo de texto llamado exactamente {legal_businessName}), un campo único por variable, sin duplicados, y sin bucles ni inversos (esos solo funcionan en Word). Las listas (apoderados, accionistas) se configuran como campos indexados por instancia fija (key_person_1, key_person_2, ... según la numeración correcta). Úsalo cuando Stage 5 lo invoque para producir el entregable en PDF. NO decide qué variable va dónde (viene cerrado de Stage 3/4) — recibe un mapeo de inserción ya resuelto y nombra/crea los campos de formulario preservando el diseño visual del PDF del cliente.
---

# Helper — Configurador de Plantillas PDF (formulario / AcroForm)

Configuras la plantilla PDF editable del cliente nombrando sus **campos de formulario** con las variables de Trébol, **sin tocar el diseño visual**. El cliente espera su PDF idéntico a la vista, solo que sus campos ahora se llaman como las variables y la plataforma los podrá autollenar.

**No redactas ni decides mapeos.** Recibes de Stage 5 un mapeo de inserción cerrado (`{field_id, location, variable | texto_fijo | vacío}`) y las instancias por lista que el usuario confirmó en Stage 4. Tu trabajo es ejecución técnica fiel.

## La regla central del PDF (interiorizar antes de empezar)

En PDF **no hay motor de plantillas con bucles**. El reemplazo se hace campo por campo de formulario. Hay **dos convenciones** y debes **replicar la que ya trae la plantilla del cliente**:

- **Convención A — token en el VALOR del campo (confirmada en producción, la más común):** el token `{variable}` se escribe como **valor / texto por defecto** del campo de formulario; el **nombre del campo es arbitrario** (`Texto100`, etc.) y se deja como está. Es la que usa el template real de Actinver (ver `grounding/example-actinver-mapping.md`). Si el cliente entrega un PDF con campos sin tokens, usa esta: pon el token como valor de cada campo.
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

### Campo simple
Renombra el campo de formulario existente (o el destinado a ese dato) para que su nombre sea exactamente la variable: `{tax_businessTaxId}`. Sin espacios dentro de las llaves, minúsculas correctas, llaves balanceadas. Conserva posición, tamaño, fuente y propiedades visuales del campo.

### Listas (apoderados, accionistas, consejo)
- Expande a tantas instancias como confirmó el usuario, con la **numeración correcta por familia** (apoderados 1-based; accionistas/consejo/administradores/comisarios 0-based).
- Si el PDF del cliente ya trae filas/campos para N instancias, nómbralos en orden. Si trae menos campos que instancias necesarias, repórtalo (no agregues campos que descuadren el diseño sin confirmar).
- Cada subcampo de cada instancia es su propio campo único: `{key_person_1_name}`, `{key_person_1_role}`, `{key_person_2_name}`, …

### Casillas / poderes (x_mark)
- Si el diseño usa casillas, configúralas como campos cuyo nombre es la variable `*_x_mark` correspondiente, una por casilla, respetando la numeración 1-based de apoderados.

### FUENTE y condicionales
- En PDF no hay inverso. La FUENTE se configura como campo(s) de texto cuyo nombre es la variable de la fuente; si el dato no existe, el campo simplemente queda vacío al llenarse. Si el diseño del cliente requiere un texto alternativo de "no encontrado", eso es captura/diseño del cliente, no un inverso — déjalo como esté salvo que el mapeo indique otra cosa.

## Preservación del diseño (sagrado)

No muevas, redimensiones ni reestilices ningún campo o elemento visual. No cambies fuentes, colores, posiciones, ni el contenido estático del PDF. **No apliques brand guidelines de Trébol.** Lo único que cambia es el **nombre interno** de los campos de formulario (y, si el diseño ya lo contemplaba, la cantidad de instancias de una lista).

## Campos SIN_VARIABLE
- **Texto fijo:** si se decidió texto fijo y el PDF lo permite, déjalo como contenido estático o campo con valor por defecto, según corresponda al diseño.
- **En blanco:** deja el campo sin nombre de variable (o como campo de captura manual del cliente). No le pongas un nombre de variable inventado.

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
