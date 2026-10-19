# WAHA (WhatsApp HTTP API)

API REST autoalojada (Docker) que envuelve varios motores de WhatsApp Web y expone una sola interfaz REST + webhooks. Cambiar de motor se hace con una variable de entorno.

Docs oficiales: <https://waha.devlike.pro/docs/>

## Motores (engines)
| Motor | Cómo funciona | Notas |
|---|---|---|
| **WEBJS** | Navegador real (Chromium) vía Puppeteer | Es el **default** si no se define `WHATSAPP_DEFAULT_ENGINE`. Más pesado (~535 MiB en una medición de terceros). |
| **WPP** | Navegador (Chrome) vía Puppeteer | Similar a WEBJS. |
| **NOWEB** | WebSocket directo, Node.js/TypeScript, sin navegador | Más liviano (~342 MiB en la misma medición). |
| **GOWS** | WebSocket directo en Go | Más nuevo; se perfila como reemplazo de NOWEB. Único con soporte de passkey. |

- Las respuestas y payloads **pueden diferir entre motores**: probar todo antes de cambiar.
- Conflicto de fuentes: la doc oficial dice que todos los motores están en Core y Plus; una página de terceros (Railway) dice que GOWS y varias sesiones simultáneas son de Plus. **Verificar la imagen vigente.**
- Habilitar `tagsEventsOn` en WEBJS puede afectar rendimiento/estabilidad.

## Sesiones
Una **sesión** = una cuenta de WhatsApp (número). El `name` la identifica (típico: `default`).

### Estados (`session.status`)
`STOPPED` → `STARTING` → `SCAN_QR_CODE` → `WORKING`. También `FAILED` y, solo en GOWS, `PASSKEY_REQUIRED` / `PASSKEY_CONFIRMATION_REQUIRED`.

- `FAILED` (autorización vencida o dispositivo desconectado): primero *restart*; si no, *logout* y luego *start*.
- QR: el primero dura 60 s, los siguientes 20 s, máximo 6; luego la sesión pasa a `FAILED`. Cada evento `SCAN_QR_CODE` implica pedir el QR de nuevo.

### Endpoints
| Acción | Endpoint |
|---|---|
| Listar | `GET /api/sessions` (`?all=true` incluye las detenidas) |
| Obtener | `GET /api/sessions/{name}` |
| Crear | `POST /api/sessions` (con `"start": false` queda en `STOPPED`) |
| Actualizar | `PUT /api/sessions/{session}` (config completa) |
| Iniciar / detener / reiniciar | `POST /api/sessions/{name}/start` · `/stop` · `/restart` |
| Logout | `POST /api/sessions/{name}/logout` (borra credenciales, conserva config) |
| Eliminar | `DELETE /api/sessions/{session}` |
| Info de cuenta | `GET /api/sessions/{session}/me` |
| QR | `GET /api/{session}/auth/qr` (`Accept: application/json` para base64) |
| Código de emparejamiento | `POST /api/{session}/auth/request-code` con `{"phoneNumber": "..."}` |

- start/stop/restart son idempotentes; stop no hace logout.
- Para no re-escanear el QR en cada arranque hay que **persistir el almacenamiento de sesiones** en un volumen.
- WAHA reinicia las sesiones que estaban activas al reiniciar; desactivar con `WAHA_WORKER_RESTART_SESSIONS=False`.

### Ejemplo: crear sesión con webhook
```json
{
  "name": "default",
  "config": {
    "metadata": { "user.id": "123" },
    "webhooks": [
      {
        "url": "https://<tu-receptor>/webhook",
        "events": ["message"],
        "hmac": { "key": "<CLAVE-SECRETA>" },
        "retries": { "policy": "constant", "delaySeconds": 2, "attempts": 15 }
      }
    ]
  }
}
```

## Enviar mensajes
Endpoints `POST` (siempre llevan `session` y `chatId` en el body):
- Texto: `/api/sendText` (admite `reply_to`, `mentions`, `linkPreview`).
- Multimedia: `/api/sendImage`, `/api/sendFile`, `/api/sendVoice`, `/api/sendVideo`, `/api/sendSticker` (archivo por `url` o `data` base64 + `mimetype`).
- Otros: `/api/sendLocation`, `/api/sendContactVcard`, `/api/sendList`, `/api/send/buttons/reply`, `/api/forwardMessage`, `/api/reaction`, `/api/star`.
- Lectura/presencia: `/api/sendSeen`, `/api/startTyping`, `/api/stopTyping`.
- Editar/borrar: `PUT` / `DELETE /api/{session}/chats/{chatId}/messages/{messageId}`.

Ejemplo:
```json
{ "session": "default", "chatId": "573001234567@c.us", "text": "Hola" }
```

### Formato de `chatId`
- Persona: número internacional sin `+` + `@c.us` (ej. `573001234567@c.us`; el número es un marcador).
- Grupo: `@g.us`. Canal: `@newsletter`. Estados: `status@broadcast`. Existen ids ocultos `@lid`.
- No usar `@s.whatsapp.net` al enviar.
- Imágenes: JPEG recomendado. Notas de voz: OGG/Opus (con `convert: true` WAHA convierte).

## Instalación
- `docker run` del Quick Start sirve para desarrollo, **no para producción**.
- Producción: Docker Compose (control total), o paneles como EasyPanel o Coolify. Añadir API key, volumen de sesiones y reverse proxy HTTPS (ver [seguridad](WAHA-Seguridad-y-Antibloqueo.md)).
- Imágenes: `devlikeapro/waha` (Core) y `devlikeapro/waha-plus`.
- Dashboard en el puerto 3000 (en el compose de ejemplo de n8n: `admin/admin`, **cambiarlo**).

## Relacionado
[Evolution API](Evolution-API.md) · [Migración desde Evolution](Migracion-Evolution-a-WAHA.md) · [Webhooks de WAHA](WAHA-Webhooks.md) · [Seguridad y antibloqueo](WAHA-Seguridad-y-Antibloqueo.md) · [WAHA + n8n](WAHA-con-n8n.md) · [Índice](Wiki-WhatsApp-Automatizacion.md)
