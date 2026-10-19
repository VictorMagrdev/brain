# WAHA: seguridad y antibloqueo

Fuentes: <https://waha.devlike.pro/docs/how-to/security/> y <https://waha.devlike.pro/docs/overview/how-to-avoid-blocking/>.

## Seguridad de la API
- **No exponer WAHA a internet** sin protección: API key + firewall.
- **API key:** variable `WAHA_API_KEY` con `sha512:{HASH}` (recomendado, solo se guarda el hash) o texto plano. Los clientes la envían en el header `X-Api-Key`; sin clave válida responde 401.
  - Generar: `uuidgen | tr -d '-'` y luego hash SHA-512.
  - En las notas usar marcadores: `<API-KEY>`.
- **Dashboard y Swagger:** `WAHA_DASHBOARD_USERNAME` / `WAHA_DASHBOARD_PASSWORD` y `WHATSAPP_SWAGGER_USERNAME` / `WHATSAPP_SWAGGER_PASSWORD`. Protegerlos no protege la API.
- **Excepciones:** `WAHA_API_KEY_EXCLUDE_PATH` (ej. `health`, `ping`); `WHATSAPP_API_KEY_EXCLUDE_PATH` para media.
- `?x-api-key=` en la URL funciona para media en el navegador, pero no poner ahí claves de admin.
- **Keys API:** claves admin (todas las sesiones) y de sesión (una sola, con `actions` acotadas: read, send, control, setting, app, delete).
- **Webhooks:** usar HMAC (ver [WAHA: webhooks](WAHA-Webhooks.md)).
- **HTTPS:** terminar en un reverse proxy (Nginx). La opción HTTPS integrada está obsoleta.
- `WAHA_NO_API_KEY=True` desactiva la seguridad: solo si se entiende el riesgo.

## Antibloqueo de WhatsApp
### Reglas generales
1. Responder solo a mensajes recibidos; no iniciar conversaciones.
2. Para que te escriban primero, compartir un enlace `wa.me` con texto predefinido.
3. Nada de mensajes repetidos o no solicitados.
4. No enviar muchos mensajes seguidos.
5. Evitar contenido sensible o prohibido; revisar las políticas de WhatsApp.

### Flujo recomendado al responder
1. `POST /api/sendSeen` (marcar visto).
2. `POST /api/startTyping` y esperar un tiempo aleatorio según la longitud.
3. `POST /api/stopTyping`.
4. `POST /api/sendText`.

### Límites a contactos nuevos
- **Error 463 (reachout timelock):** restricción temporal para escribir a contactos nuevos. Pausar esos envíos.
- **Error 475 (message capping):** cuota por ciclo; vigilar `FIRST_WARNING` / `SECOND_WARNING` y bajar el ritmo.
- En ambos casos **no reiniciar, cerrar sesión ni re-emparejar**: no acelera la recuperación y la sesión sigue en `WORKING`.

### Buenas prácticas
- Contenido real y solicitado, no marketing masivo.
- Variar y personalizar el texto (nombre).
- Horarios no fijos y pausas aleatorias (ej. 30–60 s entre mensajes).
- Agrupar contactos por código de área.
- Número con foto, nombre y estado.
- Enlaces con HTTPS y sin reputación de spam.
- Lotes con pausas (ej. máx. 4 mensajes por contacto que respondió en una hora, y parar una hora).
- Primer mensaje corto y único.
- Si te reportan como spam unas 5–10 veces, hay riesgo de bloqueo.

## Riesgo de fondo
WAHA usa WhatsApp Web, que Meta no soporta oficialmente para automatización. Un número puede ser bloqueado. Para producción empresarial crítica, evaluar la API oficial de WhatsApp Business.

## Relacionado
[WAHA](WAHA.md) · [Webhooks: conceptos](Webhooks-Conceptos.md) · [Índice](Wiki-WhatsApp-Automatizacion.md)
