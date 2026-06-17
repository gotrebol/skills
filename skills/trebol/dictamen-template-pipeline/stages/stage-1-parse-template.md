---
name: stage-1-parse-template
description: Primera etapa del pipeline de configuración de plantillas de dictamen. Lee la plantilla en blanco del cliente (Word .docx o PDF editable) y, si existe, el ejemplo ya diligenciado, y los normaliza a una estructura interna uniforme de "campos del cliente" (etiqueta literal, ubicación, si es campo único o fila repetible de tabla, formato esperado), detectando país/jurisdicción, tipo de sociedad y formato de entregable. Úsalo cuando el orquestador dictamen-template-pipeline lo invoque como primer paso, o cuando el usuario pida "entender", "parsear" o "leer" una plantilla de dictamen antes de configurarla. Es el único stage que toca los archivos crudos.
---

# Stage 1 — Parse Template (Entender la plantilla)

Primera etapa. Convierte la plantilla cruda del cliente en una estructura que el resto del pipeline consume sin ambigüedad. Es el único stage que abre los archivos del usuario; los demás trabajan sobre su output.

## Responsabilidad

**Una sola cosa:** transformar la plantilla cruda → lista normalizada de **campos del cliente**. No mapea variables (eso es Stage 2), no opina (eso es Stage 3). Solo lee, detecta estructura y estructura.

Si el input no es procesable, no lo arregla — detiene el pipeline y pide clarificación.

## Inputs

- **Plantilla del cliente** (obligatoria): `.docx` o PDF editable, en blanco (sin tokens de Trébol todavía).
- **Ejemplo diligenciado** (opcional): una plantilla ya llena con datos reales/ejemplo. Si existe, registra para cada campo qué valor de ejemplo contiene — esto le da a Stage 2 una pista de altísima confianza sobre qué variable corresponde.

## Skills de referencia obligatorios

Antes de escribir código, lee según el tipo:
- `.docx` → `/mnt/skills/public/docx/SKILL.md` si está disponible en el entorno (Claude.ai); si no, usa `python-docx` directamente para extraer texto y tablas, o desempaca el ZIP del `.docx` para inspeccionar el XML.
- PDF → `/mnt/skills/public/pdf/SKILL.md` y `/mnt/skills/public/pdf-reading/SKILL.md` si están disponibles; si no, usa `pdfplumber` o `pypdf` para inspeccionar campos de formulario AcroForm (`reader.get_fields()`).

## Proceso

### Paso 1 — Detectar tipo de archivo y editabilidad
- `.docx`: editable. Extrae texto/tablas con `extract-text`; si necesitas ubicación exacta, desempaca el XML.
- `.doc` legacy: conviértelo a `.docx` primero (ver docx SKILL).
- **PDF**: determina si es un PDF con **campos de formulario** (AcroForm) o un PDF plano/escaneado.
  - Con campos de formulario → editable, procesable.
  - Plano/escaneado (sin campos) → **no procesable para PDF**. Reporta al usuario: para PDF se necesita un formulario con campos de texto (Adobe Acrobat → Preparar formulario), o entregar la configuración como Word. No intentes inyectar campos por OCR.
- Protegido con contraseña → detente, pide la versión sin protección.

### Paso 2 — Copiar a workspace
Copia el original a `/home/claude/ws/` antes de inspeccionar. Nunca trabajes sobre `/mnt/user-data/uploads/` (read-only y debe quedar intacto).

### Paso 3 — Extraer los campos del cliente
Un "campo del cliente" es cada lugar donde el dictamen espera un dato que Trébol podría llenar. Señales:
- **Word:** etiqueta seguida de espacio/celda vacía ("DENOMINACIÓN O RAZÓN SOCIAL: ____"), celdas de tabla con encabezado y filas vacías, secciones repetibles (una fila "modelo" de apoderado/accionista/consejero que se repite).
- **PDF:** cada campo de formulario AcroForm es un campo del cliente; registra su nombre actual, tipo y página.

Para cada campo captura:
- `field_id`: id interno secuencial (`F001`, `F002`, …).
- `label`: la etiqueta literal del cliente, sin modificar (ej. "DOMICILIO FISCAL").
- `location`: ubicación exacta — Word: tabla/fila/celda o párrafo; PDF: nombre de campo + página.
- `kind`: `single` (un dato) | `table_row` (fila repetible de una lista) | `source_clause` (cláusula FUENTE/provenance) | `checkbox_like` (casilla X) | `section_header` (no es campo, es título — útil para contexto).
- `table_group`: si es `table_row`, a qué tabla pertenece (ej. "APODERADOS", "ACCIONISTAS", "CONSEJO").
- `expected_format`: `texto` | `fecha` | `numero` | `porcentaje` | `moneda` | `marca_x` | `lista`.
- `example_value`: si hay ejemplo diligenciado, el valor que aparece ahí.
- `context`: texto circundante o sección que da contexto (ayuda a desambiguar campos con etiquetas iguales en secciones distintas).

No inventes campos. No fusiones dos campos. No reformules etiquetas (Stage 5 las debe poder ubicar literalmente).

### Paso 4 — Detectar metadata global de la plantilla
- `country`/`jurisdiction`: México, Colombia, EEUU, u otro. Pistas: RFC/CURP/RPPC/notaría → México; NIT/Cámara de Comercio/RUES/DIAN → Colombia. Esto decide qué familia de variables aplica (MX vs `rut_co`/`cc_co`/`rues`).
- `entity_type` si se infiere: S.A. de C.V., S. de R.L., S.A.S., etc.
- `has_loops_capable`: Word = sí; PDF = no (registra para que Stage 2/5 sepan que las listas no se pueden expresar como bucle).
- `sections_present`: qué bloques tiene (datos generales, apoderados, accionistas, UBOs, consejo/administrador, vigilancia, consultas adicionales: RENAPO/PLD/FIEL/SIGER/INE/QR-CSF…).
- `deliverable_format`: `docx` | `pdf_form`.
- `client_name` si se infiere del archivo/encabezado; si no, `unknown`.

### Paso 5 — Validación antes de devolver
1. Hay al menos 1 campo extraído. Si 0 (o PDF sin campos), detén y reporta.
2. Cada campo tiene `field_id`, `label`, `location`, `kind`, `expected_format`.
3. `country`, `deliverable_format` y `sections_present` resueltos.
4. Las etiquetas no están truncadas ni con artefactos de parsing.

Si falla, intenta corregir una vez; si persiste, reporta exactamente qué campo/sección falló.

## Output (markdown estructurado para Stage 2)

```
# Parse Result
## Metadata
- client_name: <nombre|unknown>
- country: <México|Colombia|...>
- entity_type: <S.A. de C.V.|...|desconocido>
- deliverable_format: <docx|pdf_form>
- has_loops_capable: <true|false>
- has_filled_example: <true|false>
- sections_present: [datos_generales, apoderados, accionistas, ubos, administracion, vigilancia, consultas_adicionales, ...]
- total_fields: <N>

## Campos
### F001
- label: "DENOMINACIÓN O RAZÓN SOCIAL"
- location: Tabla 1, fila 3, celda 1
- kind: single
- table_group: null
- expected_format: texto
- example_value: "GRUPO EJEMPLO, S.A. DE C.V."   (si hubo ejemplo)
- context: "DATOS GENERALES"
### F002
...
```

## Reglas duras
1. **No mapees variables.** Eso es Stage 2.
2. **No reformules etiquetas.** Stage 5 las usa para localizar el lugar exacto donde insertar el token.
3. **No descartes campos "obvios".** Todos pasan al pipeline.
4. **No inventes metadata.** Si no infieres país, márcalo `unknown` y deja que el orquestador pregunte.
5. **Identifica filas repetibles vs campos únicos.** Confundir una fila-modelo de tabla con un campo único hace que Stage 2 proponga la variable equivocada (individual vs bucle).
6. **Trabaja sobre copia.** Nunca toques el original en uploads.

## Manejo de errores
- **PDF escaneado/sin campos:** reporta; pide PDF con formulario o entrega como Word.
- **Word con tablas muy irregulares (celdas combinadas, filas no contiguas):** haz tu mejor esfuerzo, marca campos con `extraction_confidence: low` y lista lo que ignoraste.
- **Plantilla ya parcialmente tokenizada** (el cliente metió algunas variables): regístralas como campos con `already_tokenized: true` y el token presente, para que Stage 2/3 las validen en vez de duplicarlas.
- **Idioma de la plantilla distinto del español:** procesa igual; las variables de Trébol son las mismas. Anota el idioma.

## Checklist
- [ ] Tipo y editabilidad detectados; PDF sin campos → detenido y reportado.
- [ ] Todos los campos con sus 5 atributos obligatorios.
- [ ] Filas repetibles marcadas como `table_row` con su `table_group`.
- [ ] País, formato de entregable y secciones resueltos.
- [ ] Valores de ejemplo capturados si había ejemplo diligenciado.
- [ ] No se mapeó ninguna variable ni se reformuló ninguna etiqueta.
