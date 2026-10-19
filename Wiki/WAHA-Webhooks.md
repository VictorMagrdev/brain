# WAHA: webhooks y eventos

Fuente: <https://waha.devlike.pro/docs/how-to/events/>. Conceptos generales en [Webhooks: conceptos](Webhooks-Conceptos.md).

## Dos formas de configurarlos
### 1. Por sesión (`config.webhooks` al crear/actualizar la sesión)
Cada webhook acepta: `url`, `events`, `hmac.key`, `retries` (`policy`, `delaySeconds`, `attempts`) y `customHeaders` (lista de `name`/`value`). Ver el ejemplo en [WAHA](WAHA.md).

### 2. Global (variables de entorno, aplican a todas las sesiones)
| Variable | Qué hace |
|---|---|
| `WHATSAPP_HOOK_URL` | URL del webhook |
| `WHATSAPP_HOOK_EVENTS` | Eventos separados por coma. `*` = todos (no recomendado en producción) |
| `WHATSAPP_HOOK_HMAC_KEY` | Clave HMAC |
| `WHATSAPP_HOOK_RETRIES_POLICY` | `constant`, `linear` o `exponential` |
| `WHATSAPP_HOOK_RETRIES_DELAY_SECONDS` | Retraso entre reintentos |
| `WHATSAPP_HOOK_RETRIES_ATTEMPTS` | Número de reintentos |
| `WHATSAPP_HOOK_CUSTOM_HEADERS` | `Nombre:Valor` separados por `;` |

La config global **no** aparece en `GET /api/sessions`.

## Reintentos
`constant` (mismo retraso), `linear`, `exponential` (con 20 % de jitter).

## Headers que WAHA envía
- `X-Webhook-Request-Id`: id único por petición (útil para deduplicar).
- `X-Webhook-Timestamp`: milisegundos.
- `X-Webhook-Hmac` y `X-Webhook-Hmac-Algorithm` (`sha512`): solo si hay HMAC.

## Verificar HMAC
1. Definir `hmac.key` (usar un marcador como `<CLAVE-SECRETA>` en las notas, nunca la real).
2. Calcular HMAC-SHA512 del **body crudo** con esa clave.
3. Comparar con `X-Webhook-Hmac`. Si coincide, viene de tu WAHA.

En n8n hay que activar la opción **Raw Body** del nodo Webhook para poder calcularlo (ver [n8n: nodo Webhook](n8n-Webhook.md)).

## Estructura del payload
Campos de primer nivel: `id` (ULID en minúsculas), `timestamp`, `event`, `session`, `metadata`, `me`, `payload`, `environment`, `engine`.

Evento `message`, campos de `payload`: `id`, `timestamp`, `from`, `fromMe`, `to`, `source` (`app` o `api`), `body`, `hasMedia`, `media.url` (puede ser `null` si WAHA no descargó el archivo), `ack`, `vCards`, `_data` (interno, **varía por motor**).

Tip: `fromMe` y `source` sirven para evitar bucles (que el bot responda a sus propios mensajes).

## Eventos disponibles
- **Sesión:** `session.status`, `engine.event`
- **Mensajes:** `message`, `message.any`, `message.reaction`, `message.ack`, `message.ack.group`, `message.waiting`, `message.edited`, `message.revoked`
- **Chats:** `chat.archive`
- **Grupos:** `group.v2.join`, `group.v2.leave`, `group.v2.participants`, `group.v2.participants.join-request`, `group.v2.update`
- **Presencia:** `presence.update`
- **Encuestas:** `poll.vote`, `poll.vote.failed`
- **Eventos de calendario:** `event.response`, `event.response.failed`
- **Etiquetas:** `label.upsert`, `label.deleted`, `label.chat.added`, `label.chat.deleted`
- **Llamadas:** `call.received`, `call.accepted`, `call.rejected`
- **Obsoletos:** `group.join`, `group.leave`, `state.change` (usar `group.v2.*` y `session.status`)

`message` solo trae mensajes entrantes; `message.any` incluye también los enviados por ti.

## WebSocket (alternativa)
`ws://<host>:3000/ws?session=<nombre|*>&events=<evento|*>&x-api-key=<API-KEY>`. El `*` de eventos no incluye `engine.event`.

## Relacionado
[WAHA](WAHA.md) · [Seguridad y antibloqueo](WAHA-Seguridad-y-Antibloqueo.md) · [WAHA + n8n](WAHA-con-n8n.md)
