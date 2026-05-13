# Autenticación con la API de Trébol

## URL base

```
https://api.gotrebol.com
```

Trébol usa REST + JSON. Códigos HTTP estándar.

## Header de autenticación

Todas las llamadas requieren la API key en el header `x-api-key`. **No es** `Authorization: Bearer`.

```
x-api-key: treb_sk_live_1234567890abcdef...
```

## Smoke test rápido

Para confirmar que tu API key funciona, lista las keys de tu cuenta (es una llamada de bajo riesgo):

```bash
curl -X GET "https://api.gotrebol.com/api-keys" \
  -H "x-api-key: tu_api_key"
```

Respuestas esperadas (según el OpenAPI canónico):
- `200` — la key funciona y devuelve la lista de keys de tu cuenta
- `401` — la key es inválida o se eliminó
- `404` — endpoint no encontrado (verifica la URL base)
- `500` — error interno de Trébol (reintenta o reporta)

## Gestión de API Keys

Las keys se gestionan vía dashboard o vía API.

### Vía dashboard (manual, recomendado para arrancar)

1. Login en https://app.gotrebol.com
2. Click en el ícono de ajustes (arriba derecha)
3. Selecciona **Claves secretas**
4. **Crear API Key** con un nombre descriptivo (ej. `produccion-2024`, `integracion-crm`)
5. **Copia la key inmediatamente** — solo se muestra una vez

### Vía API (programática)

| Método | Endpoint | Para qué |
|---|---|---|
| `GET` | `/api-keys` | Listar todas las keys de la cuenta |
| `POST` | `/api-keys` | Crear una nueva key |
| `DELETE` | `/api-keys/{apiKeyId}` | Eliminar una key |

### Crear key vía API

```bash
curl -X POST "https://api.gotrebol.com/api-keys" \
  -H "x-api-key: tu_api_key_existente" \
  -H "Content-Type: application/json" \
  -d '{ "id": "produccion-2024" }'
```

El campo `id` en el body es un identificador descriptivo que tú eliges (cualquier string). Trébol genera el ID interno (`ak_...`) y la key (`treb_sk_live_...`).

Respuesta `201`:
```json
{
  "id": "ak_1234567890abcdef",
  "api_key": "treb_sk_live_1234567890abcdef...",
  "created_at": "2024-01-15T10:30:00Z"
}
```

⚠️ El campo `api_key` solo aparece en esta respuesta. **Guárdalo de inmediato** en un secret manager (AWS Secrets Manager, Hashicorp Vault, 1Password, etc.).

## Códigos de error de la API de keys

| HTTP | Cuándo |
|---|---|
| `400` | Datos de entrada inválidos |
| `401` | API key inválida o no autorizada |
| `404` | API key no encontrada (en DELETE) |
| `409` | El `id` propuesto ya existe (en POST) |
| `500` | Error interno del servidor |

## Mejores prácticas

- **Nombres descriptivos**: `api-key-produccion-2024`, `api-key-webhook-notif`, `api-key-integracion-crm`. Esto ayuda cuando hay que rotar.
- **Variables de entorno**: nunca hardcodees la key. Usa `process.env.TREBOL_API_KEY` (Node) o `os.environ['TREBOL_API_KEY']` (Python).
- **Una key por entorno**: separar dev / staging / producción.
- **Rotación cada 6-12 meses** o ante sospecha de compromiso:
  1. Crear key nueva
  2. Actualizar la app
  3. Validar que todo funcione
  4. Eliminar la key vieja
- **Si una key se filtra**: elimínala YA y crea una nueva. Revisa logs por uso sospechoso.

## Ejemplo de cliente con key bien manejada

### Node.js

```javascript
const TREBOL_BASE = 'https://api.gotrebol.com';
const apiKey = process.env.TREBOL_API_KEY;

if (!apiKey) {
  throw new Error('TREBOL_API_KEY no está configurada');
}

async function trebol(method, path, body) {
  const response = await fetch(`${TREBOL_BASE}${path}`, {
    method,
    headers: {
      'x-api-key': apiKey,
      ...(body && { 'Content-Type': 'application/json' }),
    },
    ...(body && { body: JSON.stringify(body) }),
  });
  if (!response.ok) {
    const err = await response.json().catch(() => ({}));
    throw new Error(`Trébol ${method} ${path}: ${response.status} ${err.message || ''}`);
  }
  return response.json();
}

module.exports = { trebol };
```

### Python

```python
import os
import requests

TREBOL_BASE = "https://api.gotrebol.com"
api_key = os.environ.get("TREBOL_API_KEY")

if not api_key:
    raise RuntimeError("TREBOL_API_KEY no está configurada")

def trebol(method: str, path: str, body: dict | None = None) -> dict:
    headers = {"x-api-key": api_key}
    if body is not None:
        headers["Content-Type"] = "application/json"

    response = requests.request(
        method=method,
        url=f"{TREBOL_BASE}{path}",
        headers=headers,
        json=body,
    )

    if not response.ok:
        try:
            err = response.json()
        except ValueError:
            err = {}
        raise RuntimeError(
            f"Trébol {method} {path}: {response.status_code} {err.get('message', '')}"
        )

    return response.json()
```
