# Errores comunes y cómo manejarlos

## Errores HTTP del API

| `code` | HTTP | Cuándo ocurre | Acción |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Body con datos faltantes o inválidos | Revisa el `message`, corrige el payload |
| `BAD_REQUEST` | 400 | Solicitud mal formada | Revisa headers, content-type, body |
| `UNAUTHORIZED` | 401 | API key ausente o inválida | Revisa header `x-api-key`, regenera key si fue revocada |
| `FORBIDDEN` | 403 | Key sin permisos para la acción | Pide al admin que asigne permisos |
| `NOT_FOUND` | 404 | Recurso no existe | Verifica IDs (verification_id, tag, etc.) |
| `CONFLICT` | 409 | Conflicto de estado | Lee el `message` para detalles |
| `DUPLICATE_RESOURCE` | 409 | Intento de crear algo que ya existe (ej. API key con mismo `id`) | Usa otro `id` o lee el recurso existente |
| `INTERNAL_SERVER_ERROR` | 500 | Bug en Trébol | Reintenta con backoff. Si persiste, reportar a soporte |

### Estructura de respuesta de error

```json
{
  "success": false,
  "message": "El campo 'country' es obligatorio",
  "code": "VALIDATION_ERROR",
  "timestamp": "2025-01-01T12:34:56.000Z"
}
```

⚠️ Programa tu lógica con `code` y HTTP status. **No parsees el texto de `message`** — puede cambiar.

## Errores de items en webhooks

Los webhooks `verification_item.v2.completed` y `verification_item.v2.internal_status_changed` pueden incluir `item_error`:

| `item_error` | Causa | Qué decirle al usuario |
|---|---|---|
| `password_protected_pdf` | PDF protegido con contraseña | "El archivo está protegido. Súbelo sin contraseña." |
| `get_input_file_info_failed` | Trébol no pudo leer el archivo | "El archivo no se pudo procesar. Verifica que sea un PDF/imagen válido." |

### Ejemplo de manejo

```javascript
function handleItemCompleted(event) {
  const { item_error, item_type, verification_id, item_id } = event.data;
  
  if (item_error === 'password_protected_pdf') {
    notifyUser(verification_id, 'PDF protegido — sube sin contraseña');
    return;
  }
  if (item_error === 'get_input_file_info_failed') {
    notifyUser(verification_id, 'No se pudo leer el archivo — verifica el formato');
    return;
  }
  
  // Sin error, continuar
  fetchItemResults(verification_id, item_id);
}
```

## Errores de búsqueda de CURP

El webhook `verification_people.curp_search_completed` puede incluir `people_error`:

| `people_error` | Causa | Acción sugerida |
|---|---|---|
| `curp_format_error` | El CURP no cumple el formato regex de RENAPO | Pedir al usuario que corrija el CURP |
| `curp_scrapper_error` | Error extrayendo info del servicio externo | Reintentar más tarde, posiblemente RENAPO caído |
| `curp_service_unavailable` | Servicio de RENAPO no disponible | Reintentar con backoff |

## Errores específicos de items KYB

### `unknown` (clasificación falló)

Cuando subes un `generic` y Trébol no logra clasificar el documento, el item se actualiza a tipo `unknown`. Manejo:
- Detectar en webhook: `item_type === 'unknown'`
- Pedir al usuario que suba otro archivo o aclare qué documento es

### `corrupted_file` (archivo dañado)

Cuando el archivo subido no es legible, el item se actualiza a `corrupted_file`. Manejo:
- Detectar: `item_type === 'corrupted_file'`
- Pedir al usuario que vuelva a subir el archivo en buen estado

## Errores comunes de integración (no son del API)

### "Mi webhook nunca llega"

Causas posibles, en orden de probabilidad:
1. **URL no es HTTPS** → Trébol solo entrega a https://
2. **Firewall bloquea las IPs** `35.170.236.123` y `54.162.134.233`
3. **El servidor responde lento** (>30s) → timeout, Trébol asume error
4. **El webhook está desactivado** en Trébol — verificar con `GET /v2/webhooks`

### "Recibo el mismo webhook varias veces"

Es esperado. Trébol reintenta hasta 5 veces si tu endpoint no responde `2xx`. Implementa idempotencia usando la firma `v1=` del header `Trebol-Signature` como clave de deduplicación (es única por evento). Detalles y ejemplo en `flows/webhooks.md` sección "Implementa idempotencia con la firma del header" o en la [guía oficial](https://docs.gotrebol.com/guia-devs/webhooks).

### "El payload del webhook no se valida — firma inválida"

Causas comunes:
1. **Estás parseando el body como JSON antes de verificar** → tienes que validar contra el body **raw** (string original).
   - Express: `app.post('/webhook', express.raw({ type: 'application/json' }), ...)`
   - Flask: `request.get_data(as_text=True)`
2. **Estás usando comparación normal** (`===`, `==`) → usa `crypto.timingSafeEqual` (Node) o `hmac.compare_digest` (Python) para evitar timing attacks
3. **Secret incorrecto** — verifica que estás usando el secret del webhook correcto (cada webhook tiene su propio secret)
4. **Encoding** — el payload debe firmarse como UTF-8

### "401 al hacer cualquier request"

Diagnóstico:
```bash
curl -X GET "https://api.gotrebol.com/api-keys" \
  -H "x-api-key: tu_api_key" -v
```
- Si responde `401`: la key es inválida o fue revocada
- Verifica que NO uses `Authorization: Bearer` (Trébol usa `x-api-key`)
- Las keys de producción usan el prefijo `treb_sk_live_`

### "URL de mi documento da timeout"

`file_url` debe estar disponible al menos **5 minutos**. Trébol descarga el archivo, no lo proxy-ea.

Si tus URLs son privadas o efímeras (ej. URLs firmadas de S3 con TTL corto), usa el flujo de **carga directa**: Trébol te entrega un `upload_url` por item, tú subes el archivo a esa URL, Trébol procesa.

Detalle del flujo: https://docs.gotrebol.com/guia-devs/crear-verificaciones/via-api/carga-directa

## Patrón de reintento exponencial

Para errores `5xx` o de red, implementa reintentos:

```javascript
async function trebolWithRetry(method, path, body, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await trebol(method, path, body);
    } catch (err) {
      const isRetryable = err.status >= 500 || err.code === 'ECONNRESET';
      if (!isRetryable || attempt === maxRetries - 1) throw err;
      await new Promise(r => setTimeout(r, 1000 * Math.pow(2, attempt)));
    }
  }
}
```

**No reintentes errores `4xx`** — son problemas de tu request, no se van a arreglar reintentando.
