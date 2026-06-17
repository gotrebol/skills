---
name: helper-docx-template-filler
description: Helper del pipeline de configuración de plantillas de dictamen para insertar los tokens de variables de Trébol en plantillas Word (.docx) de clientes, preservando exactamente el formato original — fuentes, tamaños, colores, tablas, celdas combinadas, anchos de columna, encabezados, estilos. El cliente espera su plantilla idéntica, solo con los tokens ({legal_businessName}, bucles {#shareholders}...{/}, inversos {^...}...{/}, *_x_mark) colocados donde van. Úsalo cuando Stage 5 lo invoque para producir el entregable en Word. NO decide qué variable va dónde (eso viene cerrado de Stage 3/4) — recibe un mapeo de inserción ya resuelto y lo aplica. Su valor técnico clave: garantizar que cada token quede completo en un solo run (Word parte tokens entre runs por negritas/autocorrección, lo que rompe el reemplazo en la plataforma) y que los bucles usen variables relativas.
---

# Helper — Configurador de Plantillas Word (.docx)

Insertas los tokens de Trébol en la plantilla Word del cliente **sin tocar nada más**. El archivo de salida debe ser indistinguible del original salvo por los tokens añadidos.

**No redactas ni decides mapeos.** Recibes de Stage 5 un mapeo de inserción ya cerrado (`{field_id, location, token | texto_fijo | vacío}`) y banderas (qué filas son bucles, qué campos llevan inverso, instancias de listas). Tu trabajo es puramente de ejecución técnica fiel al formato.

## Antes de tocar nada

1. **Lee `/mnt/skills/public/docx/SKILL.md`** primero. Es obligatorio: define cómo manipular `.docx` correctamente en este entorno (librerías, rutas, cómo preservar estilos). No escribas código de Word antes de leerlo.
2. **Trabaja sobre una copia.** Nunca edites el archivo en `/mnt/user-data/uploads`. Copia a tu directorio de trabajo y opera ahí.
3. Revisa el grounding de sintaxis (`grounding/sintaxis-plantillas.md`) para el reemplazo correcto de tokens, bucles e inversos en Word.

## Reglas de inserción

### Tokens simples
Inserta el token literal exactamente como viene en el mapeo: `{legal_businessName}`, `{tax_businessTaxId}`. Sin espacios dentro de las llaves, minúsculas correctas, llaves balanceadas. Reemplaza el contenido del campo del cliente (o el placeholder/celda vacía) por el token, conservando el formato de esa celda/párrafo.

### Bucles `{#lista}...{/}` (solo Word)
- El bucle envuelve la fila o bloque que se repite. Apertura `{#shareholders}` antes del bloque repetible, cierre `{/}` (o `{/shareholders}`) después.
- **Las variables dentro del bucle son relativas:** dentro de `{#shareholders}` se usa `{name}`, `{percentage}`, **no** `{shareholder_0_name}`. Esta es la causa #1 de bucles rotos.
- En una tabla, el bucle suele envolver **una sola fila modelo**; la plataforma la repite por cada elemento. No dupliques filas manualmente.

### Inversos / condicionales `{^variable}...{/}`
- Para el patrón "no se encontró X" (FUENTE faltante, sin poderes, etc.): `{^...}texto cuando falta{/}`.
- Para FUENTE condicional, sigue el patrón documentado: bloque condicional `{#..._source_number}...{/}` + su inverso. Respétalo tal como viene en el mapeo; no lo reinventes.

### Casillas `x_mark`
- Los tokens `*_x_mark` (ej. `{key_person_1_power_administration_x_mark}`) van en la celda de la casilla correspondiente, tal como en la plantilla de referencia. Respeta la numeración del mapeo (apoderados 1-based).

## El problema crítico: tokens partidos en runs

Word divide internamente el texto en *runs* cuando hay cambios de formato (negrita, color), autocorrección, o al teclear. Un token como `{legal_businessName}` puede quedar partido en runs `{legal_` + `businessName}`, y entonces **la plataforma de Trébol no lo reemplaza** porque no lo ve como una unidad.

Por cada token insertado:
1. Asegúrate de escribirlo en **un solo run** con formato uniforme.
2. Tras escribir, **re-extrae y verifica** que el token aparezca completo y contiguo en el XML del documento (no partido entre `<w:r>`).
3. Si quedó partido, **normaliza el run** de ese token (unifica el formato del fragmento que contiene el token) y vuelve a verificar.

Esta verificación no es opcional: un token partido es un fallo silencioso que el cliente solo descubre cuando la plantilla no se llena.

## Preservación de formato (sagrado)

No cambies, en ninguna circunstancia: fuente, tamaño, color, negrita/cursiva del texto del cliente, anchos de columna, alto de fila, celdas combinadas, bordes, sombreados, encabezados, pies, numeración, ni el orden de las secciones. **No apliques los brand guidelines de Trébol** (verde, Arial) sobre la plantilla del cliente — es el formato del cliente el que manda.

El token hereda el formato del lugar donde va (si la celda era Arial 10 negro, el token va Arial 10 negro). No introduzcas formato nuevo.

## Campos SIN_VARIABLE
- Si la decisión fue **texto fijo**, escribe el texto acordado.
- Si fue **dejar en blanco**, deja la celda como está (vacía), sin token.
- Nunca metas un token inventado para "tapar" un campo sin variable.

## Output (a Stage 5)
```
{
  "success": true,
  "output_path": ".../{nombre}_configurada.docx",
  "tokens_inserted": N,
  "loops_configured": [{ "field": "F040", "loop": "shareholders", "relative_vars": ["name","percentage"] }],
  "split_tokens_fixed": [ ... ],     # tokens que quedaron partidos y se normalizaron
  "split_tokens_unresolved": [ ... ],# tokens que no se pudieron unir (a formatting_issues)
  "format_intact": true
}
```

## Reglas duras
1. **Lee el SKILL.md público de docx antes de codear.**
2. **Copia primero; nunca edites el upload.**
3. **Formato del cliente intacto; sin brand guidelines de Trébol.**
4. **Token completo en un solo run — verificar siempre.**
5. **Variables dentro de bucle = relativas.**
6. **No inventes variables ni redecidas mapeos.** Aplicas lo que viene cerrado.
7. **No dupliques filas de bucle manualmente** — el bucle envuelve la fila modelo.

## Manejo de errores
- **Token irreparablemente partido:** repórtalo en `split_tokens_unresolved`; no lo dejes silenciosamente roto.
- **Celda combinada donde el token desbordaría el formato:** inserta respetando la combinación; si no es posible sin romper diseño, reporta el campo.
- **Mapeo que llega ambiguo:** no improvises — devuelve a Stage 5 indicando qué campo no estaba resuelto.

## Gotcha de plataforma: PowerShell en Windows omite `[Content_Types].xml`

**Aplica cuando:** el entorno de ejecución es Windows y se usa PowerShell para empaquetar el `.docx` final.

**Síntoma:** el archivo se crea con el tamaño esperado pero Word lo abre como fondo gris (documento en blanco o dañado). No hay error visible durante el empaquetado.

**Causa:** PowerShell interpreta los corchetes `[` `]` como wildcards en rutas de archivo. Esto afecta a `[Content_Types].xml`, que es obligatorio en todo `.docx`. Los cmdlets `Compress-Archive` y `Copy-Item` lo omiten silenciosamente — incluso `[System.IO.File]::Exists()` devuelve `False` para ese path si se le pasa directamente desde PowerShell.

**Solución confirmada:** usar `[System.IO.Directory]::GetFiles($dir, "*", [System.IO.SearchOption]::AllDirectories)` para listar archivos (la API .NET no expande wildcards), y `[System.IO.File]::Open($path, ...)` con el path literal que devuelve `GetFiles` para leer cada uno. Nunca usar `Compress-Archive`, `Copy-Item`, ni `Get-ChildItem` para operar sobre `[Content_Types].xml` en Windows.

```powershell
# Patrón correcto para empaquetar un .docx en Windows/PowerShell
Add-Type -Assembly System.IO.Compression
$zipStream = [System.IO.File]::Create($outputPath)
$zip = New-Object System.IO.Compression.ZipArchive($zipStream, [System.IO.Compression.ZipArchiveMode]::Create)
$allFiles = [System.IO.Directory]::GetFiles($workDir, "*", [System.IO.SearchOption]::AllDirectories)
foreach ($f in $allFiles) {
    $entryName = $f.Substring($workDir.Length + 1).Replace('\', '/')
    $entry = $zip.CreateEntry($entryName, [System.IO.Compression.CompressionLevel]::Optimal)
    $es = $entry.Open()
    $fs = [System.IO.File]::Open($f, [System.IO.FileMode]::Open, [System.IO.FileAccess]::Read)
    $fs.CopyTo($es); $fs.Close(); $es.Close()
}
$zip.Dispose(); $zipStream.Close()
```

## Checklist
- [ ] Leí `/mnt/skills/public/docx/SKILL.md`.
- [ ] Trabajé sobre copia, no sobre el upload.
- [ ] Cada token insertado existe en el grounding y está bien escrito.
- [ ] Verifiqué que ningún token quedó partido entre runs.
- [ ] Bucles con variables relativas; sin filas duplicadas a mano.
- [ ] Formato del cliente intacto; sin brand guidelines de Trébol.
- [ ] SIN_VARIABLE tratado como se decidió (texto fijo / blanco).
- [ ] Output estructurado a Stage 5 con issues reportados.
- [ ] Si el entorno es Windows/PowerShell: usé `Directory.GetFiles` para empaquetar (nunca `Compress-Archive`).
