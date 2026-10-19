# n8n: nodo Webhook

Fuentes: [Webhook node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/), [Respond to Webhook](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.respondtowebhook/) y [configuración de WEBHOOK_URL](https://docs.n8n.io/hosting/configuration/configuration-examples/webhook-url/).

El nodo Webhook es un **trigger**: crea una URL que inicia un workflow y puede devolver una respuesta (sirve como endpoint de API). Conceptos generales: [Webhooks: conceptos](Webhooks-Conceptos.md).

## URLs: test vs production
| | Test | Production |
|---|---|---|
| Se registra al | pulsar **Listen for Test Event** o **Execute workflow** (workflow inactivo) | publicar/activar el workflow |
| Datos | se ven en el editor | solo en la pestaña **Executions** |

- Error típico: probar con la URL de test y olvidar activar el workflow → la URL de production no responde ("not registered").
- Fuente de terceros: el listener de test queda abierto unos ~120 s (no está en la doc oficial).

## Parámetros
- **HTTP Method:** DELETE, GET, HEAD, PATCH, POST, PUT.
- **Path:** aleatorio por defecto; se puede fijar, incluso con parámetros (`/:variable/ruta`).
- **Authentication:** Basic auth, Header auth, JWT auth o None.
- **Respond:**
  - *Immediately*: responde "Workflow got started".
  - *When Last Node Finishes*: devuelve la salida del último nodo (con **Response Data**: All Entries, First Entry JSON, First Entry Binary, No Response Body).
  - *Using 'Respond to Webhook' Node*: la respuesta la define ese nodo.
  - *Streaming response*: para nodos con streaming (ej. AI Agent).
- **Response Code:** código HTTP de éxito.

## Opciones útiles
- **Raw Body:** necesario para verificar firmas HMAC.
- **Binary Property:** guarda archivos (POST/PATCH/PUT).
- **IP(s) Allowlist:** el resto recibe 403.
- **Ignore Bots:** ignora crawlers y previsualizadores de enlaces.
- **Allowed Origins (CORS):** por defecto `*`.
- **Only Run If:** expresión sobre `$json` (`body`, `headers`, `params`, `query`); si da `false`, responde 200 sin crear ejecución.
- **Response Headers / Content-Type / Property Name / No Response Body.**

## Límites
- Payload máximo: 16 MB; en self-hosted se cambia con `N8N_PAYLOAD_SIZE_MAX`.
- Desde n8n 1.103.0 las respuestas HTML van dentro de un `<iframe>` sandbox: sin acceso a la ventana superior ni a localStorage, sin headers de auth, y las URLs relativas no funcionan (usar absolutas).

## Respond to Webhook
Requiere que el Webhook tenga *Respond = Using 'Respond to Webhook' Node*.
- **Respond With:** All Incoming Items, First Incoming Item, JSON, Text, Binary File, JWT Token, Redirect, No Data.
- Si el workflow termina sin pasar por el nodo → 200 estándar. Si falla antes → 500. Un segundo nodo Respond se ignora.
- Corre una vez con el primer ítem; para devolver varios usar *All Incoming Items* (desde 1.22.0) o un nodo Aggregate.
- Se puede activar una salida "Response" en Settings para ver lo enviado.

## Detrás de un reverse proxy (self-hosted)
n8n escucha en 5678 y el proxy expone 443, así que no puede deducir la URL pública. Se configura:
```bash
N8N_WEBHOOK_URL=https://n8n.ejemplo.com/
N8N_PROXY_HOPS=1
```
- Según la doc consultada, `WEBHOOK_URL` está obsoleta desde la 2.35.0 en favor de `N8N_WEBHOOK_URL`. Si tu instancia es más vieja, puede seguir usando `WEBHOOK_URL`: confirmar la versión.
- El último proxy debe mandar `X-Forwarded-For`, `X-Forwarded-Host` y `X-Forwarded-Proto`.
- La doc consultada no cubre timeouts del webhook; revisar la guía de configuración si hace falta.

## Relacionado
[WAHA + n8n](WAHA-con-n8n.md) · [WAHA: webhooks](WAHA-Webhooks.md) · [Índice](Wiki-WhatsApp-Automatizacion.md)
