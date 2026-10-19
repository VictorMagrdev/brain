# Evolution API

API REST de código abierto para WhatsApp y mensajería multicanal (Evolution Foundation). Nació como controlador basado en CodeChat, que usa la librería Baileys. Es lo que se usaba antes de [WAHA](WAHA.md); la guía de cambio está en [Migración Evolution → WAHA](Migracion-Evolution-a-WAHA.md).

Docs: <https://docs.evolutionfoundation.com.br/> (índice para máquinas: `/llms.txt`) · Repo: <https://github.com/EvolutionAPI/evolution-api>

> Las URLs antiguas `doc.evolution-api.com/v2/...` devolvieron 404 al investigar; la documentación se movió a `docs.evolutionfoundation.com.br`.

## Tipos de conexión
- **Baileys (WhatsApp Web):** gratis, pero depende de WhatsApp Web y puede tener limitaciones. `integration: WHATSAPP-BAILEYS`.
- **WhatsApp Cloud API (Meta):** oficial, con costos por mensaje y cumplimiento de políticas de Meta. `integration: WHATSAPP-BUSINESS`; usa número, token de acceso de Meta e ID de negocio, sin QR.

## Requisitos
- Node.js 20+, PostgreSQL o MySQL (Prisma), Redis recomendado para caché.
- Licencia Apache 2.0 con condiciones adicionales de marca (conservar logo y avisos).
- Imagen Docker: `evoapicloud/evolution-api`.
- Integraciones: Typebot, Chatwoot, Dify, OpenAI, RabbitMQ, Kafka, SQS, Socket.io, Pusher, n8n, Flowise, S3/MinIO.

## Conceptos
- **Instancia** = una conexión de WhatsApp (equivale a una *sesión* de WAHA). `instanceName` único y **no se puede cambiar**.
- **Dos niveles de clave** (header `apikey`):
  - Clave global (`AUTHENTICATION_API_KEY`): todas las instancias.
  - Token de instancia (`hash` / `hash.apikey` en la respuesta de creación): solo esa instancia.

## Endpoints de instancia
| Acción | Endpoint |
|---|---|
| Crear | `POST /instance/create` |
| Conectar / QR | `GET /instance/connect/{instanceName}` |
| Estado | `GET /instance/connectionState/{instanceName}` → `close`, `connecting`, `open` |
| Reiniciar | `PUT /instance/restart/{instanceName}` |
| Logout | `DELETE /instance/logout/{instanceName}` (conserva config) |
| Eliminar | `DELETE /instance/delete/{instanceName}` (irreversible) |
| Listar | `GET /instance/fetchInstances` |
| Ver webhook | `GET /webhook/find/{instance}` |
| Configurar webhook | `/webhook/instance` |

Patrón general: `/{recurso}/{acción}/{instanceName}`. Recursos: `/instance`, `/message`, `/chat`, `/group`, `/business`, `/call`, `/label`, `/template`, `/settings`, `/proxy`.

Cuerpo de `POST /instance/create` documentado: `instanceName` (obligatorio), `qrcode` (default `true`), `integration`, `token`, `number`, `webhook{enabled,url,events}`. La respuesta trae `instance`, `hash`, `webhook`, `settings` (rejectCall, msgCall, groupsIgnore, alwaysOnline, readMessages, readStatus, syncFullHistory) y `qrcode{pairingCode, code, base64, count}`.

## Enviar mensajes
- **Texto:** `POST /message/sendText/{instanceName}` con header `apikey`.
  - Body: `number` (con código de país, ej. `5511999999999`, sin `+` ni sufijo), `textMessage.text`, `delay`, `linkPreview`, `mentioned`, `quoted`.
  - Nota: la doc consultada muestra `textMessage.text`; en otras versiones de v2 el campo es `text` plano. **Verificar contra la versión instalada.**
- **Media:** `POST /message/sendMedia/{instanceName}`. Body `multipart/form-data` en la doc consultada: `number`, `mediatype` (`image`, `video`, `audio`, `document`), `media` (binario), `caption`, `fileName`.
- Respuesta: `key`, `message`, `status`.
- Errores: 400, 401, 403, 404, 500 con `success`, `error.code`, `error.message`.

## Webhooks
- **Por instancia** (recomendado): `/webhook/instance` con `enabled`, `url`, `webhook_by_events`, `events`, `webhook_base64`.
- **Global:** `.env` con `WEBHOOK_GLOBAL_ENABLED`, `WEBHOOK_GLOBAL_URL`, `WEBHOOK_GLOBAL_WEBHOOK_BY_EVENTS`, `WEBHOOK_EVENTS_*`.
- **Webhook por eventos:** con `webhook_by_events` se añade el evento al final de la URL en minúsculas con guiones (`MESSAGES_UPSERT` → `.../messages-upsert`).
- **Eventos (20):** APPLICATION_STARTUP, QRCODE_UPDATED, CONNECTION_UPDATE, MESSAGES_SET, MESSAGES_UPSERT, MESSAGES_UPDATE, MESSAGES_DELETE, SEND_MESSAGE, CONTACTS_SET, CONTACTS_UPSERT, CONTACTS_UPDATE, PRESENCE_UPDATE, CHATS_SET, CHATS_UPDATE, CHATS_UPSERT, CHATS_DELETE, GROUPS_UPSERT, GROUPS_UPDATE, GROUP_PARTICIPANTS_UPDATE, NEW_TOKEN.
- La doc consultada **no trae ejemplo de payload** ni documenta firma HMAC ni reintentos configurables. Capturar un payload real del ambiente antes de migrar (ver la migración).

Forma típica del payload (conocimiento general, **no confirmado en la doc**): `{ event, instance, data: { key: { remoteJid, fromMe, id }, pushName, message: { conversation }, messageType, messageTimestamp }, ... }`, donde `remoteJid` termina en `@s.whatsapp.net`.

## Variables de entorno clave
| Área | Variables |
|---|---|
| Auth | `AUTHENTICATION_API_KEY`, `AUTHENTICATION_EXPOSE_IN_FETCH_INSTANCES` |
| Servidor | `SERVER_URL`, `SERVER.PORT` (8080 por defecto), `SERVER.TYPE` |
| BD | `DATABASE_ENABLED`, `DATABASE_PROVIDER`, `DATABASE_CONNECTION_URI`, `DATABASE_CONNECTION_CLIENT_NAME`, `DATABASE_SAVE_DATA_*` |
| Redis | `CACHE_REDIS_ENABLED`, `CACHE_REDIS_URI`, `CACHE_REDIS_PREFIX_KEY`, `CACHE_REDIS_SAVE_INSTANCES`; alternativa `CACHE_LOCAL_ENABLED` |
| Webhooks | `WEBHOOK_GLOBAL_*`, `WEBHOOK_EVENTS_*` |
| Limpieza | `DEL_INSTANCE` (minutos; `false` = nunca) |
| Nombre del dispositivo | `CONFIG_SESSION_PHONE_CLIENT`, `CONFIG_SESSION_PHONE_NAME` |
| Media | `S3_ENABLED`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_BUCKET`, `S3_PORT`, `S3_ENDPOINT`, `S3_USE_SSL` |

## Relacionado
[Migración Evolution → WAHA](Migracion-Evolution-a-WAHA.md) · [WAHA](WAHA.md) · [Webhooks: conceptos](Webhooks-Conceptos.md) · [Índice](Wiki-WhatsApp-Automatizacion.md)
