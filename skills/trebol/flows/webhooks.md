# Flujo: Webhooks (notificaciones asíncronas)

## Para qué sirven

Trébol procesa documentos en background. Los webhooks te notifican cuando algo termina (item completado, verificación finalizada, CURP encontrada, etc.) sin que tengas que hacer polling.

## Crear un webhook

```
POST https://api.gotrebol.com/v2/webhooks
```

Body mínimo:
```json
{
  "url": "https://tu-app.com/webhooks/trebol",
  "events": ["verification.v2.created", "verification.v2.finished"],
  "description": "Webhook principal de producción"
}
```

Respuesta `201`:
```json
{
  "id": "wh_123",
  "url": "https://tu-app.com/webhooks/trebol",
  "events": ["verification.v2.created", "verification.v2.finished"],
  "status": "enabled",
  "secret": "whsec_XXXXXXXXXXXXXXXXXXXXXXXX",
  "created_at": "2025-08-20T10:00:00Z"
}
```

⚠️ **El `secret` solo se muestra una vez en esta respuesta.** Guárdalo de inmediato en tu secret manager.

| Endpoint | Acción |
|---|---|
| `POST /v2/webhooks` | Crear |
| `GET /v2/webhooks` | Listar |
| `GET /v2/webhooks/{webhookId}` | Obtener uno |
| `PUT /v2/webhooks/{webhookId}` | Actualizar |
| `DELETE /v2/webhooks/{webhookId}` | Eliminar |

## Tipos de eventos

### `verification.v2.created`
Verificación creada en el sistema.

### `verification.v2.finished`
Verificación completada exitosamente. **Es el evento más importante** — significa que ya puedes leer todos los resultados.

### `verification.v2.extraction_completed`
Todos los items de extracción terminaron, pero la verificación aún no está marcada como "finished". Útil para acceso anticipado a datos extraídos.

### `verification_item.v2.completed`
Un item específico completó su procesamiento. Contiene `item_error` si hubo problema:
- `password_protected_pdf` — PDF con contraseña, no se pudo procesar
- `get_input_file_info_failed` — falló al leer el archivo

### `verification_item.v2.internal_status_changed`
Cambio de estado interno de un item.

### `verification_item.v2.extraction_completed`
Item específico terminó su extracción (antes de marcarse completed). `success: boolean` indica si fue exitosa.

### `verification_people.curp_search_completed`
Trébol terminó de buscar el CURP de una persona. Posibles errores:
- `curp_format_error` — CURP mal formado
- `curp_scrapper_error` — error al extraer info del servicio externo
- `curp_service_unavailable` — servicio caído

## Ejemplos de payload por evento

### `verification.v2.finished`

```json
{
  "event_name": "verification.v2.finished",
  "data": {
    "verification_id": "5853393e-8cf7-4dc7-afd9-a92df69fff2b",
    "account_id": "212457cc-09bb-4308-b69b-f719e6f2eb03",
    "account_name": "trebol",
    "created_at": "2025-01-15T10:30:00Z",
    "status": "finished",
    "verification_tag": "tu-tag-1234"
  }
}
```

### `verification_item.v2.completed` (con error)

```json
{
  "event_name": "verification_item.v2.completed",
  "data": {
    "verification_id": "5853393e-8cf7-4dc7-afd9-a92df69fff2b",
    "item_id": 32644,
    "account_id": "212457cc-09bb-4308-b69b-f719e6f2eb03",
    "account_name": "trebol",
    "item_type": "ac_mx",
    "item_error": "password_protected_pdf",
    "completed_at": "2025-01-15T10:31:42Z",
    "verification_tag": "tu-tag-1234"
  }
}
```

`item_error` es opcional. Si el item se procesó bien, no aparece.

### `verification_people.curp_search_completed` (con error)

```json
{
  "event_name": "verification_people.curp_search_completed",
  "data": {
    "verification_id": "c0361bc7-9318-4b09-8186-4ee451cc569f",
    "item_id": 32640,
    "people_id": 3908,
    "people_error": "curp_format_error",
    "people_error_message": "CURP is required and must be a string",
    "account_name": "trebol",
    "account_id": "212457cc-09bb-4308-b69b-f719e6f2eb03",
    "verification_tag": "tu-tag-1234"
  }
}
```

## Validar la firma HMAC-SHA256

Cada webhook llega con header `Trebol-Signature: t=1640995200,v1=abc123def456...`

- `t=` — timestamp Unix de cuándo se generó la firma
- `v1=` — firma HMAC-SHA256

⚠️ Los headers HTTP son case-insensitive según el RFC. Express y la mayoría de frameworks normalizan a lowercase (`req.headers['trebol-signature']`). Si tu framework es case-sensitive, usa `Trebol-Signature` exactamente.

### Pasos para validar

1. Extraer `t` y `v1` del header
2. Construir el string a firmar: `{timestamp}.{payload_raw}`
3. Calcular HMAC-SHA256 usando tu webhook secret
4. Comparar con `v1` usando comparación de **tiempo constante** (no `===`)

### Ejemplo Node.js

```javascript
const crypto = require('node:crypto');
const express = require('express');

function verifyWebhookSignature(payload, signature, secret) {
  const elements = signature.split(',');
  const timestamp = elements.find(el => el.startsWith('t='))?.split('=')[1];
  const v1 = elements.find(el => el.startsWith('v1='))?.split('=')[1];
  if (!timestamp || !v1) return false;

  const payloadToSign = `${timestamp}.${payload}`;
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payloadToSign, 'utf8')
    .digest('hex');

  const received = Buffer.from(v1, 'hex');
  const expected = Buffer.from(expectedSignature, 'hex');
  if (received.length !== expected.length) return false;
  return crypto.timingSafeEqual(received, expected);
}

const app = express();
app.post('/webhook', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['trebol-signature'];
  const payload = req.body.toString();

  if (!verifyWebhookSignature(payload, signature, process.env.WEBHOOK_SECRET)) {
    return res.status(401).send('Invalid signature');
  }

  // Encolar para procesamiento asíncrono
  eventQueue.enqueue(payload);

  // Responder rápido (siempre 2xx si la firma es válida)
  res.status(200).send('OK');
});
```

### Ejemplo Python

```python
import hmac
import hashlib
from flask import Flask, request, abort

def verify_webhook_signature(payload: str, signature: str, secret: str) -> bool:
    elements = signature.split(',')
    timestamp = next((e.split('=')[1] for e in elements if e.startswith('t=')), None)
    v1 = next((e.split('=')[1] for e in elements if e.startswith('v1=')), None)
    if not timestamp or not v1:
        return False

    payload_to_sign = f"{timestamp}.{payload}"
    expected = hmac.new(
        secret.encode('utf-8'),
        payload_to_sign.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(v1, expected)

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('Trebol-Signature')
    payload = request.get_data(as_text=True)
    if not verify_webhook_signature(payload, signature, app.config['WEBHOOK_SECRET']):
        abort(401)
    # Encolar para procesamiento asíncrono...
    return 'OK', 200
```

## IPs de origen

Trébol envía webhooks desde:
- `35.170.236.123`
- `54.162.134.233`

Puedes whitelistear estas IPs en tu firewall, **pero siempre debes validar la firma HMAC**. Las IPs pueden cambiar; la firma no falla.

## Reintentos

Si tu endpoint no responde `2xx`, Trébol reintenta con backoff exponencial:

| Reintento | Espera | Tiempo acumulado |
|---|---|---|
| 1 | 30 s | 30 s |
| 2 | 60 s | 1.5 min |
| 3 | 120 s | 3.5 min |
| 4 | 240 s | 7.5 min |
| 5 | 480 s | 15.5 min |

Después de **5 fallos**, el webhook se marca como fallido y no se reintenta más.

Trébol reintenta cuando:
- Tu servidor responde `4xx` o `5xx`
- Hay timeout de conexión
- Error de red

## Reglas de oro

### 1. Responde `200 OK` antes de procesar

Valida la firma, encola el evento, responde. El procesamiento real va en un worker.

```text
Mal:  Procesar → guardar en DB → responder 200
Bien: Validar firma → encolar → responder 200 → worker procesa
```

### 2. Implementa idempotencia con la firma del header

Los reintentos pueden duplicar eventos. **Usa la firma `v1=` del header `Trebol-Signature` como clave de deduplicación**, ya que es única por evento. Esto es lo que recomienda la [guía oficial de webhooks](https://docs.gotrebol.com/guia-devs/webhooks).

```javascript
function getEventId(signatureHeader) {
  // Extrae la firma v1 del header Trebol-Signature
  const elements = signatureHeader.split(',');
  const v1 = elements.find(el => el.startsWith('v1='))?.split('=')[1];
  return v1;
}

const eventId = getEventId(req.headers['trebol-signature']);
if (!eventId) return res.status(400).send('Bad signature');

const ok = await redis.set(`trebol:dedup:${eventId}`, '1', 'EX', 86400, 'NX');
if (!ok) return res.status(200).send('dup'); // ya procesado
```

Almacena los IDs procesados al menos 24 horas para manejar reintentos tardíos. Redis con TTL es ideal.

### 3. No asumas orden de eventos

Puedes recibir `verification_item.v2.completed` antes de `verification.v2.created`. Si necesitas estado actual, consulta el API.

### 4. Procesa con cola asíncrona

Usa RabbitMQ, SQS, Celery, BullMQ. Si haces todo síncrono, vas a tener timeouts en picos de tráfico.

### 5. TTL de dedupe ≥ 24 h

Guarda los IDs procesados al menos 24 horas para manejar reintentos tardíos. Redis es ideal.

### 6. Verifica el timestamp para evitar replay (opcional)

Rechaza webhooks con `t=` mayor a 5 minutos en el pasado:

```javascript
const ageSeconds = Math.floor(Date.now() / 1000) - parseInt(timestamp, 10);
if (ageSeconds > 300) return res.status(401).send('Stale signature');
```

## Después de `verification_people.curp_search_completed`

Cuando recibas este webhook, puedes obtener los datos de CURP:

```
GET /v2/verifications/{verification-id}/people
```

⚠️ Path param `{verification-id}` en kebab. El field `verification_id` (snake) está en el payload del webhook, lo conviertes en path al hacer la lectura.

En la respuesta, busca la persona cuyo `people_id` coincida y lee `external_identities.curp`:

```json
{
  "external_identities": {
    "curp": {
      "success": true,
      "applicant_data": {
        "curp": "ROMC850315HDFRRS07",
        "names": "CARLOS ALBERTO",
        "first_surname": "RODRIGUEZ",
        "second_surname": "MARTINEZ",
        "birth_date": "1985-03-15",
        "gender": "HOMBRE",
        "nationality": "MEXICO"
      }
    }
  }
}
```
